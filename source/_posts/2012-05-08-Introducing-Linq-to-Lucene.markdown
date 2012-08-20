---
layout: post
title: Introducing LINQ to Lucene
comments: true
categories: 
---

_(Get the source code at [https://github.com/themotleyfool/Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq))_

For the last several weeks I've been working with [OctopusDeploy](http://octopusdeploy.com/), a new one-click deployment
system for .net applications and websites. The way you get your software into Octopus is by publishing packages to
a [NuGet](http://nuget.org/) feed. Octopus will monitor the feed, grab your package, send it out to the servers that will
host your software, then instruct the servers to unzip the package and run your deployment scripts.

We're pretty happy with Octopus, but we ran into a problem with our NuGet feed. Since we use continuous integration, we
produce 10 or 20 builds a day and each one of those successful builds becomes a release candidate that we want to be
available in Octopus. We found that as we published more packages to our internal feed, the slower Octopus became when
accessing it. It doesn't help that some of our packages are 40mb websites.

Thinking that the stateless nature of a file-based NuGet feed might be the culprit, we switched over to
[NuGet.Server](https://nuget.org/packages/NuGet.Server), an asp.net web app that provides access to NuGet packages over http.
As it turns out, NuGet.Server isn't significantly better at handling lots of packages or even a handful of rather large packages.

The reason is because NuGet.Server doesn't actually cache much information about packages, so each time a client executes a query,
the server has to:

* Enumerate all <code>*.nupkg</code> files on the file system
* Open the zip file and
    * Seek to the nuspec metadata entry
	* Parse the entry
* Obtain file metadata (last modified date, size)
* Calculate a checksum hash of the file contents (bigger package, slower hash time)

One thing NuGet.Server does that a file-based feed does not is cache some of this metadata (like the hash code), but this
is an in-memory cache, meaning that when the web app restarts or is shutdown after an idle period, the entire cache goes away.

Even with the cache, operations like finding the latest version of a package or sorting packages by name are much slower than
they could be. Not to mention, the "search" functionality leaves something to be desired.

## Hooray for Open Source ##

I set out to make NuGet.Server faster and I wanted to do it with something light weight. Something that can index and search,
but doesn't require a database and all the complexity that comes with it. If I wanted all that I could have just used
[NuGet Gallery](https://github.com/NuGet/NuGetGallery).

I decided to use [Lucene.Net](http://incubator.apache.org/lucene.net/) as my data source, but looking at the existing NuGet code,
I realized that NuGet uses [OData](http://www.odata.org) to expose a queryable data source over the web. That sounds nice and all,
but in the server side in asp.net, that basically means you need an implementation of <code>IQueryable&lt;T&gt;</code>. Those can
be hard to come by.

## Boo for OData, LINQ and IQueryable ##

Exposing a search/query interface over the web powered by Lucene is not hard to do. Exposing an OData compatible endpoint powered
by Lucene is so insanely complicated that I have to conclude that OData is a cruel joke for anyone not using Entity Framework (and
if you are using EF, the joke was already on you).

OData sounds great in that the types of queries a client can construct are limitless. But in practice, NuGet does not need this
insane amount of power. Really all that is needed is a simple interface that lets you enumerate packages in some pre-defined
ways.

But since NuGet has a large install base that is using OData, it was time to try to connect <code>IQueryable&lt;T&gt;</code> with Lucene.

There is already a [LINQ to Lucene](http://linqtolucene.codeplex.com/) project on CodePlex, but after working with it I found that
it does not support enough of the LINQ expressions to work seamlessly with OData.

So I decided to try and write my own.

## Introducing Lucene.Net.Linq ##

[Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq) is the result of my quest to make NuGet.Server faster.
The library is incomplete and the features it does support were motivated by making it compatible enough with OData in general
and NuGet.Server in particular.

This project turned out to be even more needlessly complex due to the way Microsoft's WCF Data Services [Reflection Provider](http://msdn.microsoft.com/en-us/library/dd723653.aspx)
tries to avoid <code>NullReferenceException</code> by turning simple queries into completely insane ones. This is a problem that the
NuGet Gallery folks also [ran into](http://blog.davidebbo.com/2011/08/how-odata-quirk-killed-nuget-server.html).

So a large part of Lucene.Net.Linq jumps through hoops because the Reflection Provider takes a simple OData query like:

<pre>
Packages()?$filter=startswith(Id,'foo')
</pre>

And turns it into a complex query like:

<pre>
IIF(
  (IIF(([package].Id == null), null, Convert([package].Id.StartsWith("r"))) == null),
   False,
   Convert(IIF(([package].Id == null), null, Convert([package].Id.StartsWith("foo")))))
</pre>

And hands that off to the poor IQueryable implementor to make sense of. At the end of the
day this query ends up being converted to a Lucene PrefixQuery such as:

<pre>
Id:foo*
</pre>

All of this could have been avoided if the WCF Data Services [Reflection Provider](http://msdn.microsoft.com/en-us/library/dd723653.aspx)
would let clients set [IsNullPropagationRequired](http://msdn.microsoft.com/en-us/library/system.data.services.providers.idataservicequeryprovider.isnullpropagationrequired.aspx)
to <code>false</code>.

## Conclusion ##

[Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq) was written for a very specific purpose,
but it could be generally useful for plugging Lucene into
places that already use LINQ to execute queries and to expose Lucene indexes over OData (but don't use OData; you're better than that).

We hope you find it useful!

To comply with the different licenses of Lucene.Net and [re-linq](http://relinq.codeplex.com/), the library is co-licensed under the
terms of the [LGPL](http://www.gnu.org/licenses/lgpl-2.1-standalone.html)
and [Apache License](http://www.apache.org/licenses/LICENSE-2.0).

Look for this library at NuGet.org soon!

Get the source code at [https://github.com/themotleyfool/Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq).

Up next: releasing a blazingly fast NuGet feed.
