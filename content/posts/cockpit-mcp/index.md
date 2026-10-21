---
author: Christian Goll
date: 2026-10-12 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: Cockpit MCP server
categories:
- programming
- agents
- cockpit
tags:
- programming
- agents
---

# Cockpit MCP server

The [cockpit-mcp](https://github.com/mslacken/cockpit-mcp) project builds an MCP
server on top of [Cockpit](https://cockpit-project.org/)
(https://github.com/cockpit-project/cockpit) so that an agent can drive a Linux
host the same careful way a human would use Cockpit.

## Motivation

Cockpit offers a nice way to interact with and administer your Linux system in the web browser. 
Especially for parts of the system you don't know by heart because you do not deal with 
them, Cockpit is also useful for experienced users. 
Another aspect of Cockpit is that someone designed and selected the necessary and most sensible options 
for an operation, while a standard command line tool offers **all** options. 
For example, if you want to format a disk, `gpt` is preselected and no exotic options for the 
creation of file systems, like inode size, are accessible.
This is, in my opinion, exactly what I want for an MCP server, as LLMs get sidetracked by exotic options
and tend to set them. Also, since the base training data contained a lot of old documentation, 
an LLM might choose an MBR with `ext3` if tasked with creating a filesystem.

## Implementation

Connecting to an external Cockpit server is fairly easy, as this is done via WebSocket, which is 
accessible with standard tools, and so one can fairly easily get the list of packages in Cockpit, like the file system or user management packages.
But unfortunately the easy part ends here, as the Cockpit server only delivers a set of empty `index.html` pages (one per package) plus their minified JavaScript bundles, which build the actual interface at runtime.

## Special recipes

My first idea was to parse these JavaScript pages for the "button" elements, as they will most likely be labeled
correctly and so lead to an action. For example, the "reboot" button will have the right label and effectively reboot the machine. One could then parse all these buttons into searchable MCP actions so that the LLM can trigger a reboot.
Unfortunately, in this case the "reboot" button leads to another dialog where the reboot must be confirmed.
This kind of sequence is also used for other actions.
The solution for this is to take an agent, read the Cockpit source tree, and write special recipes for all these actions
so that if one of these buttons is found, some special action is needed.
This is not an ideal solution as you need the real Cockpit source code to create all needed recipes, so this solution was discarded.

## Parse package JavaScript bundle

As an alternative you could go the other way around and start not at the trigger of the action, but check at the point
where the actions start, which is cockpit's `spawn` API. Every action that changes the host ultimately comes down to
calling `cockpit.spawn(argv, options)` over the bridge, passing the exact command and its arguments. In unminified
source that reads like
```js
cockpit.spawn(["mkfs.xfs", label, device], { superuser: "require" })
```

but the page bundle is a minified release build, so the human-readable identifiers are gone. What the parser sees is
more like
```js
cockpit.spawn(["mkfs.xfs", a, b], { superuser: "require" })
```

For the first we could simply construct an MCP server which an LLM could use; the latter (which we get in reality)
exposes the underlying command but leaves `a` and `b` as unknown holes that have to be linked back to dialog fields.
To fix this, the code can be parsed with an AST or tree-sitter (available in Python) to connect the input values to the right
input fields, which carry the right caption.
That approach works, but as the `spawn` commands often expose their underlying commands and the input values only carry a caption, the LLMs get sidetracked again or can't find the right input values.

## Conclusion

Although the basic idea was nice, it didn't work out quite well.
A real solution would be that the Cockpit authors add the MCP functions alongside when creating the Cockpit 
package and then expose them correctly.

## Final thoughts

Before the rise of LLMs this project would have failed quite early, as one would have needed to think more clearly 
on the basic idea and how to extract the data. With the help of LLMs this project is in a working state but still not really useful, as Cockpit doesn't have the correct API for this.
