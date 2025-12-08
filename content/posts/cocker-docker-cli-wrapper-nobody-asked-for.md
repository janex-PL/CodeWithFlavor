---
title: "Cocker: The Docker CLI Wrapper Nobody Asked For"
date: 2025-12-08T08:00:00+01:00
cover: 
    image: "/images/0002-cover.png"
ShowToc: true
---
Developers like to pretend we write code with precision and intention.
Reality check: half of software engineering is fixing things caused by our own mistakes.

For years, my most frequent typo has been typing `cocker` instead of `docker`.

At first I shrugged it off. Then I started getting irrationally angry about it. And then came the moment every engineer knows too well:

>Okay, I can’t fix the root problem… so I’ll build an overengineered workaround.

Welcome to the birth of **`cocker`** - a feathered, supportive, utterly unnecessary wrapper around Docker.

> Link to the repository: **[codewithflavor/cocker](https://github.com/codewithflavor/cocker)**

## The typo that wouldn’t die

This all began during a long day of container debugging — the kind where Docker errors stop being messages and start being personal attacks.

I typed `docker` dozens of times. And dozens of times, my rebellious fingers produced:

```bash
cocker ps
cocker logs
cocker compose up
```

Every. Single. Time. I had two choices:
- ~~Fix my typing~~
- Burn everything down and create a new binary called `cocker`

The choice was obvious.

## But why?

You probably have a few questions already in your mind

- Is there any value in spending time on this?
- Why don’t you just fix your typing?
- Why not just create an alias and move on with your life?

All excellent, rational questions.



## So what it does?

Basically, it's a CLI wrapper for `docker` which displays a chicken-related message with a minor roast and then executes the command with the typo fixed.

{{< figure src="/images/0002-example1.png" align=center title=">There's a chicken and a pretty awful roast!" alt="Screenshot of the terminal containing cocker command usage with an image of a chicken in ASCII display along with roast message">}}

If you have configured `docker` to be used only with root privileges and you forget about using `sudo`, additional message shows up!

{{< figure src="/images/0002-example2.png" align=center title=">Yet another bad roast!" alt="Screenshot of the terminal containing cocker command usage with an image of a chicken in ASCII display along with roast message for both typo and lack of root privileges">}}

An autocomplete script is also provided so that the user will fall into the trap of believing in usage of valid command.

{{< figure src="/images/0002-example3.png" align=center title=">Pretty sneaky!" alt="Screenshot of the terminal containing cocker auto complete usage, showing arguments from docker command">}}

## Future plans

There's still a few things left to do:

- fixing autocomplete script to handle flags and switches
- integrate with [openSUSE Build Service](https://build.opensuse.org/) for publicly available repository with `.rpm` package
- create a Powershell version to support Windows environment

Feel free to contribute to the repository!

## Lessons learned
-  Doing silly, low-stakes projects is actually fun and surprisingly refreshing.
-  GitHub Copilot works really well for Bash scripting ...
-  ... but at the same time it’s absolutely terrible at GitHub Actions.
