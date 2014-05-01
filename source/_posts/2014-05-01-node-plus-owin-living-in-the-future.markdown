---
layout: post
title: "Node + Owin: Living in the Future"
date: 2014-05-01 17:04
comments: true
categories:
---

Scott Hanselman recently blogged ["It's just a software issue"- Edge.js brings Node and .NET together on three platforms](http://www.hanselman.com/blog/ItsJustASoftwareIssueEdgejsBringsNodeAndNETTogetherOnThreePlatforms.aspx)
and to say the least I am intrigued.

I've been using Grunt (and therefore Node) to develop [Klondike](https://github.com/themotleyfool/Klondike/)
for about 6 months and it never occurred to me that hosting managed .net code
from Node was an option.

After reading Scott's post, my immediate question was if I could host an OWIN
application from Node. Some quick googling revealed [connect-owin](https://github.com/bbaia/connect-owin)
is well on its way.

It took a little elbow grease to get from the toy app samples to being able
to initialize Klondike and serve requests.

## Proof of Concept

On my Windows VM, I ran:

```
npm install connect-owin express
```

Then created this barebones Node program:

```js
var owin = require('connect-owin'),
    express = require('express');

var app = express();
app.all('/api/*', owin('bin/debug/NuGet.Lucene.Web.OwinHost.Sample.exe'));

app.listen(3000);
```

## AppDomains and Configuration

Klondike uses appSettings to configure where files are kept and which options
are enabled. It also has a fair amount of binding redirects that make sure
all the various assemblies pulled in from NuGet packages agree to use the
same version of DLL dependencies.

I realized quickly that when my assembly is loaded from Node, my config file
was being ignored.

I asked a [question](https://github.com/tjanczuk/edge/issues/131) on the Edge
Github project to confirm, but inspecting source code is appears to be the case
that when Edge loads a managed assembly, it isn't creating a full AppDomain
and specifying a config file to go with it.

To "trick" my configuration into being loaded, I copied node.exe from it's home
in Program Files to my bin/Debug directory, and then copied my config file to
`node.exe.config` in the same local directory.

Sure enough, running `.bin\Debug\node.exe server.js` happily pulled in my config settings
and binding redirects.

It would be nice if Issue 131 gets first-class support, but in the mean time
at least I know there's a way to load complex applications into Edge.

## Whither Mono?

Edge (and therefore connect-owin) have first-class support for the Mono Framework,
although unfortunately I was unable to get beyond an issue with Ninject. As it
turns out, Ninject has a separate build configuration to support Mono and the
packages on nuget.org won't work when you try to use them with Mono.

This is pretty unfortunate and I'll be looking for a new IoC container to replace
Ninject with. One that has built in integration with WebApi and SignalR would
be nice.

## The Future

The Future is now. Being able to use Bower, Grunt and Node with integrated .NET
request processing means teams can use the same great front-end tools that
are sweeping the community by storm while still building rich server-side
components powered by .NET.

I'm a huge fan of c# and .NET, but love that this enables front-end developers
who aren't to be able to contribute to these projects without forcing them to
stop using Grunt, Bower and whatever other favorite tools they bring from their
toolboxes.

Are Edge and connect-owin production ready? I don't know, but I also don't really
care. To me, these are ideal tools for development and testing, but once
`grunt build` gives me an app tree in `./dist` I'd rather just push my app to
the cloud and let someone else worry about hosting.

## Source

My efforts to convert Klondike to support OWIN self hosting are a
work-in-progress, but you can see the source code on the [webapi-oata](https://github.com/themotleyfool/NuGet.Lucene/tree/webapi-odata/source/NuGet.Lucene.Web.OwinHost.Sample)
branch of [NuGet.Lucene](https://github.com/themotleyfool/NuGet.Lucene/)
on Github.
