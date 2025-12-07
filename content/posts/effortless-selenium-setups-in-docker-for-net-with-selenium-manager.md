---
title: "Effortless Selenium Setups in Docker for .NET with Selenium Manager"
date: 2023-10-29T10:45:00+02:00
cover: 
    image: "/images/0001-cover.png"
ShowToc: true
---

I've been using Selenium in my projects for automation, web scraping and even for some minor workarounds not related to website navigation at all. However, because I've been usually deploying my .NET apps using Docker containers, I had to overcome some obstacles. 

In this article, I'll share my experiences, from handling dependencies and image sizes to navigating versioning challenges. We'll also explore Selenium Manager's latest features and its game-changing impact on Selenium + Docker + .NET integration. 

_Important note: This article assumes you are using Selenium with Chrome browser. Not every section of the article is applicable if you are using Firefox or Edge._ 

## Dependencies make the blue whale even bigger!

When we want to build a Docker image running our .NET code, we typically create a Dockerfile that carries out the following steps:

1. Copy source files to an image with .NET SDK installed
2. Restore NuGets and publish the app to a folder
3. Copy the output files into an image with .NET runtime installed

If we need to perform actions in a web browser through code, it's essential to make sure the browser, its necessary libraries, and the matching WebDriver are available. This can result in significantly larger image sizes in order to fulfil these requirements.

To demonstrate this, I've prepared three Docker images:
- Bare .NET runtime image tagged with version `6.0.22-bookworm-slim` as a base line

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:6.0.22-bookworm-slim AS base
```

- .NET runtime + required dependencies for launching Chrome browser inside container [from this list](https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md#chrome-doesnt-launch-on-linux)
```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:6.0.22-bookworm-slim AS base

RUN apt-get update \
  && apt-get install --no-install-recommends -y ca-certificates fonts-liberation libasound2 \
  libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 \
  libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 \
  libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 \
  libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils curl \
  && apt-get clean
```

- .NET runtime + required dependencies + latest stable version of Google Chrome installed
```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:6.0.22-bookworm-slim AS base

RUN apt-get update \
  && apt-get install --no-install-recommends -y ca-certificates fonts-liberation libasound2 \
  libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 \
  libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 \
  libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 \
  libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils curl \
  && apt-get clean

RUN curl -LO  https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \ 
  && apt-get install --no-install-recommends -y ./google-chrome-stable_current_amd64.deb \
  && rm google-chrome-stable_current_amd64.deb \
  && apt-get clean
```
As you can see, the Dockerfiles were written with size optimization in mind by chaining commands in a single `RUN` command in order to reduce the number of layers, installing only the necessary dependencies without any additional recommended packages and cleaning up after install using `apt-get clean` command.

After building the images, we are going to list them along with their size

```shell
REPOSITORY                         SIZE
runtime                            194MB
runtime-dependencies               310MB
runtime-dependencies-chrome        667MB
```

As you can see, the `runtime-dependencies` and `runtime-dependencies-chrome` images are 1.5x and 3x larger than the base `runtime` image, respectively.

You've probably noticed that there is a step missing in all Dockerfile which would download a ChromeDriver matching the version of installed. I haven't include this step for two reasons:
- The driver itself weighs about 15-17 MB, so it doesn't have that much impact on the image size
- There are packages providing a feature of automatic driver download during the runtime of application i.e. [WebDriverManager.Net](https://github.com/rosolko/WebDriverManager.Net) for .NET applications

## Versioning nightmare

For a long time, developers using Selenium with Chrome browser had no option to download a specific version of the browser from an official source. This could potentially cause some problems:
- you couldn't perform a rollback to the previous Chrome version in case if a critical bug was introduced in a new version, unless you've built a runtime image when the previous version was considered the latest stable and you've applied a sensible image versioning strategy i.e. by not pushing images to the registry with a `latest` tag only
- updating the version of .NET means that you are automatically going to install the new version of Chrome if a new version was released in the meantime, which is something that may not be desired
- it's hard to come up with a reasonable versioning format, because now not only you should specify the .NET version and OS used, but also a Chrome version installed. You could also use some custom versioning format, but it would be probably confusing for a typical developer that is used to the tags provided by Microsoft
- you needed also to make sure that the ChromeDriver version supports the currently installed version of Chrome

With the [release of Selenium `4.6.0`](https://www.selenium.dev/blog/2022/selenium-4-6-0-released/), Selenium team introduced Selenium Manager, which in the beginning provided automated driver management. This feature simplified the process of manually installing and guarding the versions of drivers and browsers, but the Chrome browser itself still didn't provide any versioning.

On June 12 2023, [Google announced the Chrome for Testing browser](https://developer.chrome.com/blog/chrome-for-testing/), a special flavor of Chrome dedicated for automation and testing. What's more important, the team had introduced the versioning of the browser binaries. Thanks to this change, developers could now install a specific version from `113.0.5672.0` onward.

## And then Selenium dropped the bombshell

With the introduction of Selenium `4.11.0`, Selenium team announced that [Selenium Manager now includes automated browser management](https://www.selenium.dev/blog/2023/whats-new-in-selenium-manager-with-selenium-4.11.0/) based on Chrome for Testing. Thanks to this feature, if Selenium doesn't detect the version of the browser requested by the code installed on the machine, it automatically downloads the binaries. As of the time this article was written, version `4.12.0` introduced support for Mozilla Firefox and Edge support is planned to be released with version `4.13.0`.

## Current status of Selenium + Docker

Let's analyze the benefits that all of these changes have brought us in the context of the issues mentioned earlier.

### Reduced base image size
Since we no longer need to manually install the browser and its driver, we can remove this process from our Dockerfile and leave only the installation of required dependencies. Below is the Dockerfile for .NET runtime image tagged `6.0.22-bookworm-slim ` that can be safely used with Chrome for Testing and Selenium.

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:6.0.22-bookworm-slim AS base

RUN apt-get update \
  && apt-get install --no-install-recommends -y ca-certificates fonts-liberation libasound2 \
  libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 \
  libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 \
  libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 \
  libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils curl \
  && apt-get clean
```
### Easier .NET version management
We can go one step further and create a more general version of Dockerfile, where you can provide the base image version as a build argument and specify also the default version of the image.
```dockerfile
ARG dotnet_tag=6.0.22-bookworm-slim
FROM mcr.microsoft.com/dotnet/runtime:$\{dotnet_tag\} AS base

RUN apt-get update \
  && apt-get install --no-install-recommends -y ca-certificates fonts-liberation libasound2 \
  libatk-bridge2.0-0 libatk1.0-0 libc6 libcairo2 libcups2 libdbus-1-3 libexpat1 libfontconfig1 \
  libgbm1 libgcc1 libglib2.0-0 libgtk-3-0 libnspr4 libnss3 libpango-1.0-0 libpangocairo-1.0-0 \
  libstdc++6 libx11-6 libx11-xcb1 libxcb1 libxcomposite1 libxcursor1 libxdamage1 libxext6 \
  libxfixes3 libxi6 libxrandr2 libxrender1 libxss1 libxtst6 lsb-release wget xdg-utils curl \
  && apt-get clean
```
Now, when building you can specify the runtime image version for building, but also at the same time you can use the same version to tag the new image

```shell
DOTNET_TAG=6.0
docker build --build-arg "dotnet_tag=${DOTNET_TAG}" -t runtime-chrome:$DOTNET_TAG .
```

### Managing browser version from code

Since our runtime image is no longer coupled with the specific version of the browser, we can utilize the new Selenium Manager feature to request any Chrome for Testing browser version from `113` onward.

I've created a simple program to demonstrate how you can request a specific browser version. Notice that we are not providing a specific version at compile time, which means that you can request any supported version of the browser during runtime.

```csharp
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;

var chromeOptions = new ChromeOptions();
// remember about the necessary arguments when running inside container! 
chromeOptions.AddArguments("--headless=new","--disable-gpu",
    "--no-sandbox", "--disable-dev-shm-usage"); 

while (true)
{
    Console.Write("Provide a version of Chrome browser you want to launch (type 'q' to quit):");
    var input = Console.ReadLine();
    
    if (string.Equals(input, "q", StringComparison.OrdinalIgnoreCase))
        break;

    chromeOptions.BrowserVersion = input;

    var chromeService = ChromeDriverService.CreateDefaultService();
    chromeService.SuppressInitialDiagnosticInformation = true;

    try
    {
        var driver = new ChromeDriver(chromeService,chromeOptions);
        
        driver.Manage().Timeouts().ImplicitWait = TimeSpan.FromSeconds(5);

        driver.Navigate().GoToUrl("https://www.whatsmybrowser.org/");
        Console.WriteLine(driver.FindElement(By.TagName("h2")).Text);
        
        driver.Quit();
    }
    catch (Exception e)
    {
        Console.WriteLine(e);
    }
}
```
Next, we are going to publish our program as a Docker image using the Dockerfile below
```dockerfile
#Change the default runtime to our custom base image
FROM runtime-chrome:6.0 AS base 
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["SeleniumChromeDocker.csproj", "./"]
RUN dotnet restore "SeleniumChromeDocker.csproj"
COPY . .
WORKDIR "/src/"
RUN dotnet build "SeleniumChromeDocker.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "SeleniumChromeDocker.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SeleniumChromeDocker.dll"]
```
After building, we can run this image in interactive mode and check if we can request any valid version of Chrome for Testing browser
```shell
docker build -t selenium-chrome-docker .
docker run --rm -it selenium-chrome-docker
```
Below is an example output. When providing browser version, we can use various different formats such as whole version number, major number or even release channels.
```
$ docker run --rm -it selenium-chrome-docker
Provide a version of Chrome browser you want to launch (type 'q' to quit):114
ChromeDriver was started successfully.
You’re using Headless Chrome 114.
Provide a version of Chrome browser you want to launch (type 'q' to quit):115
ChromeDriver was started successfully.
You’re using Headless Chrome 115.
Provide a version of Chrome browser you want to launch (type 'q' to quit):116.0.5800.0 
ChromeDriver was started successfully.
You’re using Headless Chrome 116.
Provide a version of Chrome browser you want to launch (type 'q' to quit):stable
ChromeDriver was started successfully.
You’re using Headless Chrome 118.
```

## Summary
In my opinion, Selenium Manager is a great tool that will vastly improve the dev experience when working with Selenium. Thanks to this tool, working with web browsers has become less problematic and the application is easier to containerize. I'm also glad that Google has finally introduced more accessible version management with the release of their new Chrome flavor.

If you want to test this integration on your own, checkout my [GitHub example repository](https://github.com/codewithflavor/DotnetSeleniumDockerRuntimeExample) where I've gathered all source code used in this article along with updates.