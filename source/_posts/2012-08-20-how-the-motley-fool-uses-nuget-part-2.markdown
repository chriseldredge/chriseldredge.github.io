---
layout: post
title: "How The Motley Fool Uses NuGet (Part 2)"
date: 2012-08-20 13:56
comments: true
categories: 
---

[Last time](/blog/2012/08/07/how-the-motley-fool-uses-nuget/)
I talked about how my development team progressed from having all
of our .net code in a single repository with a single solution to using a more
modular architecture complete with encapsulated domains.

When we started using this appraoch, we were still limited in a few ways:

* Everyone needs to integrate with the newest code
* Difficult to patch an old version of a dependency
* Cascading failures on the build server

Even though we broke the ProjectReference rats nest, we still had an
implicit dependency on various shared code. It all had to be checked out
and built in the right order.

## Binary Package Management ##

The next logical step was to further decouple our shared code by
packaging it up and publishing those packages. If we could do that,
we could decide when to upgrade dependencies on a product by product
basis.

There are two package managers in the .net ecosystem: OpenWrap and NuGet.

When we started shopping around, OpenWrap had been around longer and seemed
to be a better choice. There's a [comparison of the products](http://stackoverflow.com/questions/4256994/openwrap-vs-nuget)
on stackoverflow.

We worked with OpenWrap for over 6 months and during that time started
to find some problems around integration with Visual Studio and ReSharper.
OpenWrap wants to manage dependencies per solution, and we have many cases
where we want to control dependencies at a per project level. We also
started to notice that NuGet was getting new versions released on a fairly
regular schedule, while OpenWrap 2.0 was in unstable beta limbo for over
a year.

Around the same time we started playing with [Octopus Deploy](http://octopusdeploy.com/)
for deploying our code. Since Octopus uses NuGet packages for deployment, we
figured it would make sense to standardize on one package management system
for both deployments and dependency management. It's true that those are
separate problem spaces, but having less build scripts is always a good thing.

## Thoughts on NuGet ##

### Conventions ###

NuGet has several conventions that make it easy to create simple packages
that others can reference. You can share assemblies and content easily,
and when you want to customize anything there are some powershell extension
points you can hook into.

One problem we run into is that when building packages, sometimes there's
a NuGet convention we want to customize or suppress, and often we can't.

For example, if you create a `nuspec` and place it adjacent to a `csproj`
file, NuGet will look at the project and automatically inject metadata
and content into the package. For some things, you can override this
behavior with explicit specifications in the nuspec, but the behavior
can be surprising and confusing.

### Dependency Scoping ###

NuGet supports the concept of transitive dependencies... sort of. If you
install package A, and A depends on package B, NuGet will go find a version
of B and install it while installing A. However, NuGet doesn't do any record
keeping to remember that B is a transitive dependency. To your project, A and
B appear simply as direct dependencies.

There may be cases where A depends on B at runtime, but consumers of A
shouldn't need to code against B at design time.

There may be other cases where B is an optional dependency for A, and
A can be used without it.

Since NuGet doesn't have a concept of scope, it only has one simplistic
approach to dealing with transitive dependencies: treat them just like
direct dependencies.

### Upgrade Behavior ###

When you ask NuGet to update a specific package, it will first look for
updates to transitive dependencies that the package depends on. This may
seem obvious or desirable to some, but personally I find it confusing.
You can control this behavior with the `-IgnoreDependencies` flag in
the Package Management Console, but oddly you don't get that option
in the command line `nuget.exe` or from the Visual Studio GUI Package
Manager.

### Package Feed Performance ###

We use continuous integration, and every successful build produces
"release candidate" versions of packages. We generate 50 to 100 packages
a day.

Using the simple NuGet UNC share quickly failed to scale, so next we tried
NuGet.Server and found that it doesn't perform well either.

NuGet Gallery seemed like overkill with its SQL Server requirement, so
I started optimizing NuGet.Server. This project ended up taking quite
a while, but the good news is that the fruits of the labor are now
open source on GitHub at [https://github.com/themotleyfool/NuGet](https://github.com/themotleyfool/NuGet).

For more information about that project, see my [previous post](/blog/2012/07/03/Speeding-Up-NuGet-Server/).

### Refactoring Applications and Shared Code ###

We try to use [Semantic Versioning](http://semver.org/) to communicate
breaking changes in the packages we publish, so sometimes when
we want to use a refactoring tool like Change Method Signature
or Use Base Class it would be nice to have application and shared
code loaded into a single instance of Visual Studio.

We created a tool called [SlimJim](https://github.com/TheMotleyFool/SlimJim)
that generates these Solution files on the fly.

If you create a Solution with application code and shared library code,
ReSharper will be smart enough to apply refactoring tools across the projects
even though ProjectReference style references are not being used.

However, Visual Studio won't know the correct order to build projects in,
and won't automatically copy outputs from shared libraries over to applications.

We extended SlimJim to convert assembly references to project references and back
to address this limitation.

## Conclusion ##

In terms of capability and maturity, we're in a much better place than
we were a few years ago. However, we still have a ways to go in terms of
productivity and workflow.

NuGet has helped us move in the right direction and we hope to see
further enhancements and even contribute some more of our own as we
develop them.
