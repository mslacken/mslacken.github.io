---
author: Christian Goll
date: 2026-09-16 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: Using bubblewrap with agents
categories:
- programming
- agents
- containers
tags:
- programming
- agents
---

# Using bubblewrap to contain agents

Using coding agents like [`opencode`](https://github.com/anomalyco/opencode), [`claude`](https://github.com/anthropics/claude-code), or [`gemini-cli`](https://github.com/google-gemini/gemini-cli) makes programming with LLMs incredibly powerful. However, this comes with a security cost: you have to trust the sandboxes of these agents, and even if you fully trust them, the underlying LLM might attempt to access your git tree or other parts of your filesystem. It gets even worse because these agents often run on JavaScript/Node.js or use npm-like mechanisms to pull in untrusted packages.

The ideal solution would be to spin up a virtual machine, copy the project code into it, and avoid running the agent directly on your host machine inside your home directory. Since this still comes with a significant setup cost, a lighter approach is to leverage containerization techniques instead.

The tool [`bubblewrap`](https://github.com/containers/bubblewrap), which is part of [Flatpak](https://flatpak.org/), does exactly this by running binaries inside a private namespace. Within this private namespace, we can mount only the strictly necessary file paths for the agent, leaving out sensitive directories like `~/.ssh` or `~/.config`. While [`ai-bwrap`](https://github.com/didvc/ai-bwrap) provides a convenient way to do this, it originally left out one important detail: the private namespace still contained the bind-mounted current working directory with full read and write access, meaning the agent could still mess up your codebase!

I have resolved this by adding the `--overlay` option, which mounts the current working directory using the kernel's [OverlayFS](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html). This means the agent has full read and write access to the directory, but any modifications live in a temporary layer on top of the actual directory. After the agent exits, **you** can decide whether to apply those changes or simply discard them.


## Example session

An example session now looks like this:
```
ai-bwrap opencode --overlay --ro-bind ~/bin/
ai-bwrap: overlay scratch: /home/chris/.local/share/ai-bwrap/run.932629
                                   ▄     
  █▀▀█ █▀▀█ █▀▀█ █▀▀▄ █▀▀▀ █▀▀█ █▀▀█ █▀▀█
  █  █ █  █ █▀▀▀ █  █ █    █  █ █  █ █▀▀▀
  ▀▀▀▀ █▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀ ▀▀▀▀

  Session   Proofreading ai-bwrap blog post
  Continue  opencode -s ses_f54d04cdcffeEIou7uTG33Yo23

ai-bwrap: agent exited rc=0
ai-bwrap: changes made in the overlay:
    A  .git/opencode
    M  _posts/2026-09-16-ai-bwrap
ai-bwrap: write these changes back to /home/chris/programming/github/mslacken/mslacken.github.io? [y/N] y
ai-bwrap: wrote changes back to /home/chris/programming/github/mslacken/mslacken.github.io
```
