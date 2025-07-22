---
layout: post
title: "Oops! Copilot deployed to prod. Be careful with your extensions and MCP servers"
description: "Came across an interesting issue recently where I asked Copilot to change my MSSQL extension connection to a different database then asked it to run some queries only realize they ran against the wrong database."
date: 2025-07-22T12:30:00-07:00
tags: T-SQL
image: img/postbanners/2025-07-22-oops-copilot-deployed-to-prod.jpg
---

It's been nearly a year since my last blog post, so I thought I'd try to come back with a somewhat easy one. AI assisted development tools have really come a long way the last few years and it's only going to get crazier. Unfortunately right now, we're still in that awkward phase where we're trying to figure out what works well, what doesn't, and how all the different features and pieces will work together.

Well, a few days ago, I ran into the result of one of those awkward pieces when combining the MSSQL extension for VS Code, MSSQL MCP Server and Copilot.

The short of it is...I asked Copilot to change the connection used by the MSSQL extension to use a particular database. I later asked Copilot to describe a table in the database (which uses the MSSQL MCP server), only for it to claim the table didn't exist. I realized right away it was due to competing connections between the MSSQL extension and the MSSQL MCP Server configuration. It was also at that moment where I realized this situation could potentially be SO MUCH worse than simply not finding a table...

So let's set up a worst case scenario and see what happens.

---

## Setting up the environment

To recreate this issue we have a few dependencies that need to be set up:

* [VS Code](https://code.visualstudio.com/download)
* [GitHub Copilot - Agent mode](https://code.visualstudio.com/blogs/2025/04/07/agentMode)
* [MSSQL extension for VS Code](https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-extension-visual-studio-code)
* 2 SQL Server databases 
* [MSSQL MCP Server](https://github.com/Azure-Samples/SQL-AI-samples/tree/main/MssqlMcp/dotnet) - I'm not going to walk you through setting up the MCP server. At this point, it's a very manual process, but it's all documented in this link.

Get VS Code installed, install the MSSQL extension, set up GitHub Copilot and Copilot agent mode.

You'll need two SQL Server databases to set up this example, I personally just run SQL Server in a docker container for testing like this. (Pro tip - [You can use the MSSQL VS Code extension to set up local SQL Server containers in just a few clicks](https://learn.microsoft.com/en-us/sql/tools/visual-studio-code-extensions/mssql/mssql-local-container)).

I created two new databases. One named `Development` and one named `Production`...I wonder where this is going 🤪.

```tsql
CREATE DATABASE Production;
CREATE DATABASE Development;
```

I then set up two new connections in the MSSQL VS Code extension - Make sure you configure the database on the connection itself (this is important).

![asdf asdf asdf asdf](/img/oopscopilot/20250722_004254.jpg)

And finally, it's time to set up the MCP server connection. In this case, I'm going to use a connection string for the `Production` database:

```json
"MSSQL MCP": {
  "type": "stdio",
  "command": "C:\\tools\\SQL-AI-samples\\MssqlMcp\\dotnet\\MssqlMcp\\bin\\Debug\\net8.0\\MssqlMcp.exe",
  "env": {
    "CONNECTION_STRING": "Data Source=localhost;Initial Catalog=Production;User ID=sa;Password=yourStrong(!)Password;Trust Server Certificate=True"
  }
}
```

## Let's deploy to prod by accident on purpose

We're finally ready to cause some problems. By this point you should have everything set up and ready to go...VS Code, Copilot, Agent Mode, two databases to play with, MSSQL Extension with a database connection configured for each database and the MSSQL MCP Server configured to point at the production connection string.

Open up a new Copilot chat in VS Code and set it to Agent mode (only Agent mode has access to "tools" like MCP servers). Then make sure you have the MSSQL Extension and MSSQL MCP Server tools selected for Copilot to have access. Do this by clicking on the "Configure Tools" icon:

![A screenshot from VS Code of the Copilot prompt text box set to use Agent mode and an arrow pointing at the Configure Tools wrench icon.](/img/oopscopilot/20250722_010759.jpg)

Ensure both tools are showing in this list and enabled for Copilot...(Don't forget to click OK at the top...That messes me up every time)

![A screenshot from VS Code of the MCP tools drop down menu showing all tools checked and enabled for the MSSQL MCP server as well as the MSSQL extension.](/img/oopscopilot/20250722_010735.jpg)

If you don't see these, then you need to go back and figure out what you haven't set up yet.

Now let's go about our day as an AI leveraging database developer. First lets ask Copilot to set our connection to use the development database:

![A screenshot from VS Code starting off a chat conversation asking Copilot to connect to the development database.](/img/oopscopilot/20250722_012455.jpg)

Great!

So to explain what just happened...we asked Copilot to connect to the development database. It analyzed the list of tools we've made available to it and it determined that we're likely asking to change our MSSQL extension connection. So it asked the extension to list all available connections, it reviewed that list, saw the connection named "Development" and asked the extension to connect to it.

What comes next is where we get into the confusing bits...

Let's have a nice little chat with Copilot. We'll ask it to create a new table and verify the table exists...

![A screenshot from VS Code showing the full conversation with Copilot. Asking it to change connection to development. It confirms this is done. Then asking it to again confirm which database we are connected to, and it again says the Development database.](/img/oopscopilot/20250722_013337.jpg)

I don't trust it, so lets check it ourselves via SSMS...

![A screenshot from SSMS querying the sys.tables view in both Production and Development databases. The results show the new table that was created only exists in Production despite Copilot saying it was deployed to Development.](/img/oopscopilot/20250722_014155.jpg)

Uh oh...That's weird...why did it deploy that to Production even though we confirmed multiple times that it was deployed to Development? Is Copilot lying? No, it's not. Technically this is user error. But the point of this exercise is to show how easy it could be to run something in Production while Copilot is 100% confident that it was run in Development.

The reason this happened is because of how we configured the MCP server earlier with the Production connection string.

The problem is that Copilot is unaware of what happens within an MCP server, nor is it aware of the configuration settings. We asked Copilot to change our local connection in VS Code, so it knew to use the MSSQL Extension tools for this request. But when we asked it to create a new table in said database...The only tool we've enabled that can serve that request is the the MCP server. The downside is, the MCP server connection is configured via the main MCP server configuration, in this case, the Production database.

Unfortunately, Copilot has no idea what's going on inside of an MCP server. All it knows about is the output provided back to it and in the case of creating our table, the MCP server simply returned a success message. It had no idea the server actually connected to an entirely different database.

## Moral of the story?

Mind your P's and Q's. As long as the MCP server requires a hard-coded connection string for its connection, this problem is going to exist and it's going to pop up. I wouldn't be surprised if this hasn't already caused some problems.