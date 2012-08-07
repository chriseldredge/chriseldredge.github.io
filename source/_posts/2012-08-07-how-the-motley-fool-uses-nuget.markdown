---
layout: post
title: "How The Motley Fool uses NuGet (part 1)"
date: 2012-08-07 14:25
comments: true
categories: 
---

This post describes how we came to using binary package management. In the next part I'll get into NuGet.

In the beginning, there was one repository and it held all the projects for The Motley Fool, and it was good. 
There were around a dozen asp.net web projects, a smattering of service and console apps, and a bunch of class libraries
to hold shared code. There was one Solution (sln) to rule them all.

As time went on, we found that there are downsides to the one-giant-solution approach to .net development:

* [Big Ball of Mud](http://www.laputan.org/mud/)
* Slow builds
* Tight coupling
* Configuration hell
* Hard to release different applications on different schedules

Typically our larger applications would be split into several projects following a typical N-tier layered architecture:

* Web
* Service
* Domain
* Data Access

Despite our attempts to encapsulate data access and domain logic behind the service project, code ended up leaking out
to the point where domain projects were using types and methods from unrelated domain projects. Cats and dogs were
sleeping together.

Around this time Steven Bohlen presented a [talk](http://unhandled-exceptions.com/blog/index.php/2010/11/27/dc-altnet-presentationthats-a-wrap/)
to the Washington DC Alt.NET User Group titled "Domain Driven Design Implementation Patterns in .NET". While some of us
were already familiar with concepts of DDD, this talk lit a spark for us to try fixing our big ball of mud.

In late 2010 we started to make some changes. Instead of having one giant repository, shared code would be split out
into separate repositories. We also took this opportunity to introduce a new project organization and architecture.

We established one repository to hold utility code, broken into specific class libraries:

* <tt>Fool.Abstractions</tt> - similar in spirit to System.Web.Abstractions; adds interfaces and wrappers to various FCL types that lack them
* <tt>Fool.Lang</tt> - similar in spirit to [Jakarta Commons Lang](http://commons.apache.org/lang/); adds general utility classes and methods not found elsewhere
* Other projects that extend 3rd party class libraries to make them easier for us to work with in standardized ways.

Then we established another repository to hold Domain Driven, er, Domains. For example, many of our applications and web sites deal with
stock market data, so one of our business domains is Quotes. In the Quotes Domain we have these projects:

* <tt>Fool.Quotes</tt> - contains service interfaces and value types; serves as an API to the domain
* <tt>Fool.Quotes.Core</tt> - contains domain logic, models, and entities; serves as a private implementation
* <tt>Fool.Quotes.Web.Api</tt> - exposes Fool.Quotes interfaces over a RESTful web API

The key to keeping our domains distinct and decoupled is to keep Core projects private. While Core is required at runtime,
it should never be referenced at compile time. To bridge the gap, we use Dependency Injection to provide concrete implementations.

Domains may depend on other domains provided that they consume each other through the API project. That way entities and business logic
are kept focused on their own concerns and don't leak out to other problem areas where they don't fit.

## Gluing It Together ##

Having projects split into different repositories and different solutions meant that we couldn't simply have one mega Solution
that includes everything. That's by design, so good on us. But this introduces a problem in that we still need to reference code
from our utility projects and DDD projects in our applications. The first solution we came up with to handle this problem was
to use the [AssemblyFolders](http://msdn.microsoft.com/en-us/library/wkze6zky.aspx) registry to have our libraries
appear in the <tt>Add Reference</tt> dialog. Then to solve the runtime dependency on our private Core assemblies, we install
those to the GAC so they can be loaded using reflection by our IoC container.

This approach worked fine, mostly. But we encountered some downsides after using it for a while:

* Need to have all library code checked out and built on each development machine
* No built-in way to manage different versions of the same dependency
* [GAC considered harmful](http://www.sellsbrothers.com/Posts/Details/12503)
* Hard to debug build errors and runtime errors

Using Continuous Integration means we're producing new builds dozens of times a day, so it isn't practical for us to
manage different assembly versions for each build. Like most shops, we leave our assembly versions at 1.0.0.0 despite
injecting actual version information into the AssemblyInformationalVersion attribute.

In order to support parallel development, we needed to find a more flexible way of managing dependencies, and
at this point we started to look at binary package management.
