---
title: "Changing Version Managers"
subtitle: ""
date: 2022-01-17T07:45:00-06:00
tags: ["rvm", "nvm"]
---

Ever since macOS Big Sur, launching Terminal, more specifically a shell, for the first time takes a significant amount of time to load. One of the things I did in an attempt to mitigate this slow startup time was to switch to ZSH, but that only made a marginal improvement. Looking deeper, it appears that upon every reboot, the cache used by macOS to know where all of the developer tools are (used by xcodebuild) is invalidated and recreated. Therefore, if you are using one of those tools (e.g. git) upon shell startup, you are impacted by this process.

Simple fix then, right? Almost. By adjusting shell startup to remove dependencies on developer tools, I still noticed a sizable delay. This lead me down the rabbit hole of what else I was loading into a new shell: [RVM](https://rvm.io/) & [NVM](https://github.com/nvm-sh/nvm).

Both RVM and NVM override shell commands to do their work (e.g. cd), as such, that involves a lot of overhead when these scripts get loaded into the shell interpreter. I confirmed that the delay went away after removing them on initialization and even added back the dependency on developer tools. However, I still need the functionality that these tools provide. I then went on a search to find more performant replacements and ended up using [rbenv](https://github.com/rbenv/rbenv) and [Volta](https://volta.sh/).

rbenv does not have gemset functionality like RVM, but I'll accept the loss of that feature as an actual benefit due to the fact that I don't need to have more storage allocated to duplicate gems. Volta, on the other hand, does function similarly to RVM by having node and its dependencies separated by project. The big win though is that the use of both of these tools do not require overriding shell commands, rather, using small and performant shims on the PATH.

---

I do wish that Apple would improve the process by which the developer tool cache is managed as this wasn't a problem in previous macOS versions.
