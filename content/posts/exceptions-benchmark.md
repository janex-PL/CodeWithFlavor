---
title: "How expensive is throwing exceptions in .NET - and does it actually matter?"
date: 2026-07-09T12:00:00+02:00
cover:
  image: "/images/0003/cover.png"
ShowToc: true
math: true
---

If you've been developing in .NET for a while, you've probably come across the **`Result` type**: an error-as-value representation of either success or failure, often modeled as a discriminated union in languages that support them directly.

Developers working with functional programming languages typically take discriminated unions for granted, using them extensively during their work. In contrast, developers using object-oriented languages such as C# are often introduced to `Result` types as another programming pattern that can be used instead of exceptions.

One common claim you'll encounter in articles, courses and online discussions is:

> `Result` types improve application performance, because **throwing exceptions is expensive**.

That made me wonder:

> Just **how expensive** are exceptions in .NET, exactly?

## Theory time!

When an exception is thrown, the code path typically consists of the following steps:

1. The exception object needs to be **constructed**, just like every other object
2. The runtime **captures stack trace information** during the throw.
3. Runtime determines which catch block to execute by **walking up the call stack** until it finds a matching handler
4. If the exception occurs within a `try` block that has associated `finally` blocks, the runtime **executes them in order** as the stack unwinds

Now let's compare this with a list of steps executed when returning a `Result` value representing failure, or another error value:

1. Failure / erroneous object is **constructed**
2. Created object is **returned from the method**

Without going in depth on how each of the steps are performed under the hood, we can see that handling exceptions is a more complex procedure than handling a failure object.

Based on that, we should expect error handling with a `Result` type to be faster than throwing and catching exceptions. Let's see if the benchmarks can confirm that!

## Test environment

We are going to run benchmarks on the following machine:

- **OS**: `openSUSE Tumbleweed, version 20260715`
- **Kernel**: `7.1.3-1-default`
- **CPU**: `Intel Core i5-10310U, 4 cores / 8 threads`
- **RAM**: `16 GB DDR4`
- **Runtime**: `.NET 10.0.8`

As for the `Result` type used in benchmarks, we are going to use two implementations:

- `ClassResult<T>`: reference type, implemented as a `sealed class`

```csharp
public sealed class ClassResult<T>
{
    public bool IsSuccess { get; }
    public T? Result { get; }
    public string? Error { get; }

    private ClassResult(bool isSuccess, T? result, string? error)
    {
        IsSuccess = isSuccess;
        Result = result;
        Error = error;
    }

    public static ClassResult<T> Success(T? result) => new(true, result, null);
    public static ClassResult<T> Failure(string? error) => new(false, default, error);
}
```

- `StructResult<T>`: value type, implemented as a `readonly record struct`

```csharp
public readonly record struct StructResult<T>
{
    public bool IsSuccess { get; }
    public T? Result { get; }
    public string? Error { get; }

    private StructResult(bool isSuccess, T? result, string? error)
    {
        IsSuccess = isSuccess;
        Result = result;
        Error = error;
    }

    public static StructResult<T> Success(T? result) => new(true, result, null);
    public static StructResult<T> Failure(string? error) => new(false, default, error);
}
```

For the sake of simplicity, both implementations are using `string` as the error representation.

## Object creation benchmark

Let's start with a simple benchmark to compare the time it takes to create `Result` types and exceptions.

```csharp
[MemoryDiagnoser]
public class ObjectCreationBenchmark
{
    private Consumer _consumer = null!;

    [GlobalSetup]
    public void Setup()
    {
        _consumer = new Consumer();
    }

    [Benchmark(Baseline = true)]
    public void Exception()
    {
        var exception = new Exception("Failed!");
        _consumer.Consume(exception);
    }

    [Benchmark]
    public void Result_Class()
    {
        var result = ClassResult<int>.Failure("Failed!");
        _consumer.Consume(result);
    }

    [Benchmark]
    public void Result_Struct()
    {
        var result = StructResult<int>.Failure("Failed!");
        _consumer.Consume(result);
    }
}
```

Here are the results:

| Method        |           Mean |          Error |         StdDev | Ratio | RatioSD |   Gen0 | Allocated | Alloc Ratio |
| ------------- | -------------: | -------------: | -------------: | ----: | ------: | -----: | --------: | ----------: |
| Exception     | 12.282&nbsp;ns | 0.2443&nbsp;ns | 0.2285&nbsp;ns |  1.00 |    0.03 | 0.0383 |     120 B |        1.00 |
| Result_Class  |       8.802 ns |      0.0959 ns |      0.0897 ns |  0.72 |    0.01 | 0.0102 |      32 B |        0.27 |
| Result_Struct |       1.423 ns |      0.0197 ns |      0.0184 ns |  0.12 |    0.00 |      - |         - |        0.00 |

Since both `Result` implementations are lightweight, it is not surprising that creating a `Result` is faster and allocates less memory than creating an `Exception`.

> The `struct`-based `Result` type is particularly interesting because, in this benchmark, it avoids heap allocations and ends up being the most efficient option. That does not automatically mean it is always the best choice, though.
>
> Struct-based results can avoid allocations, but larger structs can increase copying costs. They also need careful API design to avoid accidental boxing or default-value confusion.
>
> Personally, I'm against choosing a solution purely for performance reasons while ignoring the purpose it serves in the design. From a design perspective, `struct` is best suited for representing a value made up of multiple fields, but one that does not carry enough meaning in the domain to justify modeling it as a business concept with a class.
>
> There are also other factors to consider beyond raw performance numbers. You can find more about the tradeoffs and recommended approach regarding `struct` vs `class` in the [framework design guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct).

## Error consumption benchmark

Next, we will see how the performance compares when actually handling these errors.

For this benchmark, I'm going to define some test methods used to benchmark error consumption on multiple levels of nested calls.

```csharp
public class NestedErrorMethods
{
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static int ExceptionLevel10() => throw new Exception("Failed!");
    // chaining calls from level 10 to level 1...
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static int ExceptionLevel1() => ExceptionLevel2();

    private static readonly Exception StaticException = new("Failed!");
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static int StaticExceptionLevel10() => throw StaticException;
    // chaining calls from level 10 to level 1...
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static int StaticExceptionLevel1() => StaticExceptionLevel2();

    [MethodImpl(MethodImplOptions.NoInlining)]
    public static ClassResult<int> ClassResultLevel10() => ClassResult<int>.Failure("Failed!");
    // chaining calls from level 10 to level 1...
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static ClassResult<int> ClassResultLevel1() => ClassResultLevel2();

    [MethodImpl(MethodImplOptions.NoInlining)]
    public static StructResult<int> StructResultLevel10() => StructResult<int>.Failure("Failed!");
    // chaining calls from level 10 to level 1...
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static StructResult<int> StructResultLevel1() => StructResultLevel2();
}
```

- `[MethodImpl(MethodImplOptions.NoInlining)]` attribute was used to prevent the JIT from inlining the call chain and making the benchmark for nested calls redundant
- In order to show the cost of throwing and handling exception without the object creation overhead, an additional scenario using pre-initialized exception was included

> Reusing a static exception is **not a recommended production technique**. It is included only to isolate part of the cost by removing the explicit `new Exception(...)` allocation from the benchmarked method.
>
> Exceptions carry throw-specific state, especially stack trace information, so static exception reuse should be treated only as a synthetic benchmark case.

### Single call benchmark

We are going to start with measuring error consumption for a single call

```csharp
[MemoryDiagnoser]
public class ErrorConsumingBenchmark
{
    private Consumer _consumer = null!;

    [GlobalSetup]
    public void Setup()
    {
        _consumer = new Consumer();
    }

    [Benchmark(Baseline = true)]
    public void Exception()
    {
        try
        {
            var result = NestedErrorMethods.ExceptionLevel10();
            _consumer.Consume(result);
        }
        catch (Exception e)
        {
            _consumer.Consume(e);
        }
    }

    [Benchmark]
    public void Exception_Static()
    {
        try
        {
            var result = NestedErrorMethods.StaticExceptionLevel10();
            _consumer.Consume(result);
        }
        catch (Exception e)
        {
            _consumer.Consume(e);
        }
    }

    [Benchmark]
    public void Result_Class()
    {
        var result = NestedErrorMethods.ClassResultLevel10();
        if (!result.IsSuccess)
        {
            _consumer.Consume(result.Error!);
        }
    }

    [Benchmark]
    public void Result_Struct()
    {
        var result = NestedErrorMethods.StructResultLevel10();
        if (!result.IsSuccess)
        {
            _consumer.Consume(result.Error!);
        }
    }
```

Here are the benchmark results:

| Method           |              Mean |           Error |          StdDev | Ratio | RatioSD |   Gen0 | Allocated | Alloc Ratio |
| ---------------- | ----------------: | --------------: | --------------: | ----: | ------: | -----: | --------: | ----------: |
| Exception        | 2,160.399&nbsp;ns | 42.2216&nbsp;ns | 59.1889&nbsp;ns | 1.001 |    0.04 | 0.0992 |     320 B |        1.00 |
| Exception_Static |      2,023.167 ns |      39.7514 ns |      69.6213 ns | 0.937 |    0.04 | 0.0610 |     200 B |        0.62 |
| Result_Class     |          8.654 ns |       0.2077 ns |       0.3294 ns | 0.004 |    0.00 | 0.0102 |      32 B |        0.10 |
| Result_Struct    |          2.747 ns |       0.0863 ns |       0.1027 ns | 0.001 |    0.00 |      - |         - |        0.00 |

Based on the results, we can see that throwing an exception causes additional memory allocation and increased execution time.

During a `throw`, the runtime captures stack-trace-related information and stores it on the exception. The human-readable `StackTrace` string is typically formatted later, but the throw still pays for capturing the diagnostic state needed to produce it.

For the `Result` objects we can still see low or no memory allocation, and minimal execution-time overhead.

### Nested call benchmark

In the previous benchmark, the call stack was shallow. Let's try to simulate a more realistic scenario, where errors are created deep in the call stack.

We will reuse the same benchmark from the previous section, but this time calling `...Level1` methods, which are chained with 10 other method calls.

For this scenario, the benchmark results are as follows:

| Method           |             Mean |           Error |          StdDev | Ratio | RatioSD |   Gen0 | Allocated | Alloc Ratio |
| ---------------- | ---------------: | --------------: | --------------: | ----: | ------: | -----: | --------: | ----------: |
| Exception        | 5,577.84&nbsp;ns | 107.262&nbsp;ns | 127.688&nbsp;ns | 1.001 |    0.03 | 0.4196 |    1336 B |        1.00 |
| Exception_Static |      5,556.92 ns |       95.169 ns |       84.365 ns | 0.997 |    0.03 | 0.3815 |    1216 B |        0.91 |
| Result_Class     |         24.11 ns |        0.434 ns |        0.406 ns | 0.004 |    0.00 | 0.0102 |      32 B |        0.02 |
| Result_Struct    |         23.34 ns |        0.371 ns |        0.347 ns | 0.004 |    0.00 |      - |         - |        0.00 |

From this benchmark, we can observe even greater increase of memory allocation and execution time during exception throwing, even though the call stack is still relatively simple compared to real-life scenarios, where exceptions can be thrown from an even deeper and more complicated call stack, sometimes being also wrapped by other exceptions with their own properties and separate call stack!

> `NoInlining` makes the call-chain cost visible and keeps the nested benchmark meaningful. In real application code, some of these simple `Result` methods could be inlined, so the absolute numbers should not be treated as universal.

On the other hand, `Result` objects are still allocating same amount of memory and execution time increase is negligible. Of course, that is not guaranteed since `Result` classes could be implemented with more complex error objects, introducing more overhead, but still, we can more or less expect what would be the potential sizes of those structures.

## Successful path benchmark

So far, we have focused on the performance cost of failure scenarios. However, we should expect our logic to run successfully most of the time. Let's see if there are any performance costs when **there are no errors to handle.**

For this benchmark, we are going to define some simple methods, that are going to return successful values and results.

```csharp
public class SuccessfulMethods
{
    [MethodImpl(MethodImplOptions.NoInlining)]
    public static ClassResult<int> SuccessfulClassResult() => ClassResult<int>.Success(SuccessValue());

    [MethodImpl(MethodImplOptions.NoInlining)]
    public static StructResult<int> SuccessfulStructResult() => StructResult<int>.Success(SuccessValue());

    [MethodImpl(MethodImplOptions.NoInlining)]
    public static int SuccessValue() => 42;
}
```

In this benchmark, we will try to see if introducing any form of error handling incurs any performance cost if there are no errors to handle.

```csharp
[MemoryDiagnoser]
public class SuccessfulPathBenchmark
{
    private Consumer _consumer = null!;

    [GlobalSetup]
    public void Setup()
    {
        _consumer = new Consumer();
    }

    [Benchmark(Baseline = true)]
    public void Plain()
    {
        var result = SuccessfulMethods.SuccessValue();
        _consumer.Consume(result);
    }

    [Benchmark]
    public void Exception()
    {
        try
        {
            var result = SuccessfulMethods.SuccessValue();
            _consumer.Consume(result);
        }
        catch (Exception e)
        {
            _consumer.Consume(e);
        }
    }

    [Benchmark]
    public void Result_Class()
    {
        var result = SuccessfulMethods.SuccessfulClassResult();
        if (result.IsSuccess)
        {
            _consumer.Consume(result.Result);
        }
    }

    [Benchmark]
    public void Result_Struct()
    {
        var result = SuccessfulMethods.SuccessfulStructResult();
        if (result.IsSuccess)
        {
            _consumer.Consume(result.Result);
        }
    }
}
```

Here are the results of the benchmark:

| Method        |          Mean |          Error |         StdDev | Ratio | RatioSD |   Gen0 | Allocated | Alloc Ratio |
| ------------- | ------------: | -------------: | -------------: | ----: | ------: | -----: | --------: | ----------: |
| Plain         | 1.278&nbsp;ns | 0.0496&nbsp;ns | 0.0464&nbsp;ns |  1.00 |    0.05 |      - |         - |          NA |
| Exception     |      1.348 ns |      0.0444 ns |      0.0416 ns |  1.06 |    0.05 |      - |         - |          NA |
| Result_Class  |      8.167 ns |      0.1979 ns |      0.3414 ns |  6.40 |    0.34 | 0.0102 |      32 B |          NA |
| Result_Struct |      2.843 ns |      0.0864 ns |      0.1267 ns |  2.23 |    0.12 |      - |         - |          NA |

In this benchmark, adding a `try/catch` block has no meaningful measurable cost on the successful path (the difference between `Plain` and `Exception` methods is within the error margin), compared to methods using `Result` pattern, where all values, successful or not, are wrapped using our custom type.

In larger methods, exception-handling regions may still influence JIT optimizations, but the main cost of exceptions remains throwing them, not having a `try/catch` block.

## Exceptions vs Result break-even point

Given the results from benchmarks, we can attempt to calculate a break-even point of success to failure ratio, where the exception-based approach only would be more performant overall than `Result`-based approach and vice versa.

Let's define the following variables for calculations

- $A$ represents exception-based method
  - $A_S$ represents execution time for returning success, while $A_F$ represents execution time for returning failure
- $B$ represents `Result`-based method
  - $B_S$ represents execution time for returning success, while $B_F$ represents execution time for returning failure
- $S$ represents number of invocations returning success
- $F$ represents number of invocations returning failure

The total execution times are:

$$
  T_A = SA_S + FA_F
$$

$$
T_B = SB_S + FB_F
$$

The break-even point is where both methods take the same total time. After some rearrangement we can calculate **success-to-failure ratio**:

$$
r = \frac{S}{F} = \frac{B_F-A_F}{A_S-B_S}
$$

For an easier interpretation, we can make slight adjustments in order to calculate **success probability**:

$$
 p = \frac{S}{S+F} = \frac{B_F-A_F}{(A_S-B_S)+(B_F-A_F)}
$$

Using the single-call benchmark and comparing exception-based handling with the class-based `Result`, we get:

$$
p \approx \underline{\underline{99.684 \\%}}
$$

Since we are using mean values, a confidence interval calculated using `Error` values from benchmarks would provide more precise answer. To be honest, though, the exact value is not that important in order to show **how quickly the exceptions are getting expensive**.

With these specific benchmark numbers, if failures occur more often than roughly $0.316\\%$ of calls, the class-based `Result` approach becomes faster overall. If failures are rarer than that, the exception-based approach can be faster purely from a throughput perspective.

Please keep in mind that this number is not universal. It changes depending on whether the `Result` is a `class` or `struct`, how deep the stack is, and how expensive the success and failure paths are in the real application.

## Conclusions

At this point, my curiosity has been satisfied. These simple, _microbenchmarks_ and some basic math should provide enough information to answer the question asked at the beginning of the article:

> **Throwing exceptions truly is expensive. Much more expensive than using `Result` objects for expected failures.**

But... I don't feel comfortable leaving this statement as-is. There are definitely some caveats to that.

### Exceptions are actually good for truly exceptional cases

A common misuse of `Result` is wrapping every method in `try/catch` and converting all exceptions into failure results.

```csharp
public async Task<Result<CustomerDto>> GetCustomer(Guid id)
{
    try
    {
        var customer = await _dbContext.Customers.FindAsync(id);

        if (customer is null)
            return Result.Failure<CustomerDto>("Customer not found");

        return Result.Success(new CustomerDto(customer.Id, customer.Name));
    }
    catch (Exception ex)
    {
        return Result.Failure<CustomerDto>($"Unexpected error: {ex.Message}");
    }
}
```

This usually does not improve the design - it just hides exceptional situations behind a different API shape.

If a method can legitimately return "not found", "validation failed", or "conflict detected", `Result` can express that clearly. But if the database connection dies, the ORM throws because of misconfiguration, or the code hits an unexpected null, those are still exceptions, not business-level results.

In such cases, exceptions should be allowed to bubble up the call stack instead of being mechanically converted into `Result` values. They signal that an exceptional case occurred and preserve diagnostic information that is useful for debugging. Good luck troubleshooting those cases without a captured call stack.

### The purpose of the `Result` pattern is to model business errors

`Result` pattern is a missing building block in cases where developers want to describe and handle errors strictly related to business logic.

This can be especially useful when we introduce a custom `Error` class and use it as a base type of errors in `Result` object implementation:

```csharp
public class Error
{
    public string Message { get; }

    public Error(string message)
    {
        Message = message;
    }

    public static implicit operator Error(string message) => new(message);
}

public class Result<T>
{
    // ...
    public Error? Error {get;}
    // ...
}
```

This gives us a lot of ways we can design and interact with errors:

```csharp
// simple error? no problem
return Result<int>.Failure("Something went wrong!");

// more complex errors? let's go!
public abstract class CustomerError : Error
{
    public int CustomerId { get; }

    public CustomerError(int customerId, string message) : base(message)
    {
        CustomerId = customerId;
    }
}

public class CustomerNotEligibleError : CustomerError
{
    public string Reason { get; }

    public CustomerNotEligibleError(int customerId, string reason) : base(customerId,
        $"Customer {customerId} not eligible due to {reason}")
    {
        Reason = reason;
    }
}
// and then somewhere in logic...
return Result<int>.Failure(
    new CustomerNotEligibleError(
        customerId,
        "Customer has not enough points"));
```

### Performance gain is just a side effect

Now that we have some new error types, let's try to refactor one of the common examples of exception misuse in logic control:

```csharp
public decimal CalculatePrice(Order order)
{
    try
    {
        var discount = _discountService.GetDiscount(order.CustomerId);
        return order.Total - discount;
    }
    catch (CustomerNotEligibleException)
    {
        return order.Total;
    }
    catch (CustomerBlockedException)
    {
        return order.Total + 25m;
    }
    catch (Exception)
    {
        return order.Total;
    }
}
```

The biggest concern should not be the performance, but the fact that business exceptions can be easily mixed up with application exceptions, leading to important exceptions being swallowed and not handled properly.

If we switch to `Result` pattern, we can get this code:

```csharp
public decimal CalculatePrice(Order order)
{
    var discountResult = _discountService.GetDiscount(order.CustomerId);

    if (discountResult.IsSuccess)
    {
        return order.Total - discountResult.Value;
    }

    return discountResult.Error switch
    {
        CustomerNotEligibleError => order.Total,
        CustomerBlockedError => order.Total + 25m,
        _ => order.Total
    }
}
```

Now that business logic errors are handled separately, we can more easily monitor and handle exceptional cases. The performance gain here is simply a side effect.

## Closing thoughts

Use `Result` for expected, **domain-level failures** that callers are supposed to handle. Use exceptions for unexpected, exceptional failures where stack traces and diagnostic context matter. The performance benefit of avoiding exceptions in regular control flow is real, but it is a consequence of **better error modeling** - not the main reason to adopt the pattern.
