---
author: Christian Goll
date: 2024-11-13 12:00:00+01:00
layout: post
license: CC-BY-SA-3.0
title: How to create a MCP server
categories:
- programming
- mcp
tags:
- machine learning
- programming
- mcp
---

# How to create a MCP server

This guide is for developers who want or need to build an MCP server. It describes how to implement an MCP for listing and adding users.

## What is a MCP server

A MCP server is a wrapper which sits between a LLM (Large Language Model) and an application and wraps calls from the LLM to the application in JSON.
You might be tempted to wrap existing APIs of your application via [fastapi and fastmcp](https://fastmcp.wiki/en/integrations/fastapi) but as described by [mostly harmless](https://www.jlowin.dev/blog/stop-converting-rest-apis-to-mcp), this is a bad idea.

The main reason for this is that an LLM performs text completion based on the 'downloaded' internet and can focus on the topic for not more than approximately 100 pages of text. It's hard to fill these pages with chat; you may have never encountered this limit. This also means that you have to have a user story or tasks, which has to fill this book, with all the possible failures and dead ends. In the our example it will `add a user "tux" to the system`.

The first pages of this imaginary book, are already filled by the *system prompt* and with the description of the MCP tool and its parameters. This description is provided by tool author, so you can be very descriptive when writing the tool descriptions. A few more lines of text won't harm.

Now every tool call has a JSON overlay, so you also want to avoid too many tool calls. Try to minimize the number of tools and combine similar operations into one tool. If you had a tool interacting with [systemd](https://github.com/openSUSE/systemd-mcp), you would have just one tool that combines enabling, disabling, starting, restarting... the service, and not one tool for each operation.

For the tool output, do not hesitate to combine as much information as possible. A good tool output shouldn't just return the group ID (GID) but also the group name.

The caveat here is that you can easily oversaturate the LLM with too much information, like returning the information of `find /`. This would completely fill up the imaginary book of the LLM conversation. In such a case, trim the information and provide parameters for tools, like filtering the output.

This boils down to the following points:
* Have a user story for the tools.
* Provide extensive tool descriptions and their parameters
* Condense the tools into sensible operations and don't hesitate to add many parameters
* A tool call can have several API calls
* Avoid overload: LLMs can't ignore output, so you are responsible for trimming information
And also the following bonus point, which I learned along the way:
* Avoid a `verbose` parameter; an LLM will always use it

>[!NOTE]Always remember:
>"Context is King"

