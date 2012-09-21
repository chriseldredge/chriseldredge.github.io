---
layout: post
title: "RavenDB NuGet Review"
date: 2012-09-21 15:25
comments: true
categories: 
---

[Ayende](http://ayende.com/blog) has been doing a series of posts about how simple and fast RavenDB is, using NuGet
as an example and pointing out some [complex, poorly performing queries](http://blog.nuget.org/20120824/nuget-feed-performance-update.html)
that have been bogging down nuget.org recently.

Series 1

1. [NuGet Perf Problems, part I](http://ayende.com/blog/158145/nuget-perf-problems-part-i)
1. [Nuget Perf Problem, Part II-Importing To RavenDB](http://ayende.com/blog/158177/nuget-perf-problem-part-iindash-importing-to-ravendb)
1. [NuGet Perf, Part III-Displaying the Packages page](http://ayende.com/blog/158209/nuget-perf-part-iiindash-displaying-the-packages-page)
1. [NuGet Perf, Part IV-Modeling the packages](http://ayende.com/blog/158210/nuget-perf-part-ivndash-modeling-the-packages)
1. [NugGet Perf, Part V-Searching Packages](http://ayende.com/blog/158211/nugget-perf-part-vndash-searching-packages)
1. [NuGet Perf, Part VI AKA how to be the most popular dev around](http://ayende.com/blog/158241/nuget-perf-part-vi-aka-how-to-be-the-most-popular-dev-around)
1. [NuGet Perf, Part VII AKA getting results is only half the work](http://ayende.com/blog/158242/nuget-perf-part-vii-aka-getting-results-is-only-half-the-work)
1. [NuGet Perf, Part VIII: Correcting a mistake and doing aggregations](http://ayende.com/blog/158465/nuget-perf-part-viii-correcting-a-mistake-and-doing-aggregations)

Series 2

1. [NuGet Perf, The Final Part - Load Testing - Setup](http://ayende.com/blog/158529/nuget-perf-the-final-part-ndash-load-testing-ndash-setup)
1. [NuGet Perf, The Final Part - Load Testing - The Tests](http://ayende.com/blog/158530/nuget-perf-the-final-part-ndash-load-testing-ndash-the-tests)
1. [NuGet Perf, The Final Part - Load Testing - The Results](http://ayende.com/blog/158561/nuget-perf-the-final-part-ndash-loading-testing-ndash-results)
1. [NuGet Perf, The Final Part - Load Testing - Results ^ 2](http://ayende.com/blog/158593/nuget-perf-the-final-part-ndash-load-testing-ndash-results-2)
1. [NuGet Perf, The Final Part - Load Testing - Source Code](http://ayende.com/blog/158625/nuget-perf-the-final-part-ndash-load-testing-ndash-source-code)

This series piqued my interest given that I've been working on Lucene.Net.Linq and integrating it with NuGet.Server. I haven't worked
directly with RavenDB, but I have used some products like Octopus Deploy which uses RavenDB. It seems like a pretty cool product and
I don't have anything against it. However, there are some problems with the blog series that might lead the casual reader
to be misled about what has been accomplished, especially with regards to declarations of victory and some misleading
performance numbers.

I know the blog series is only meant to show off RavenDB and give potential users an idea of what their code might look like,
but nonetheless some hefty claims are made about performance in comparison to the Entity Framework / SQL Server solution in
use at [nuget.org](http://nuget.org/). Let's take a look.

## Pick a Schema, Any Schema Will Do ##

Of course if you're going to model data in a new persistence layer, it makes perfect sense to simplify or improve the
way your data is stored and indexed for later retrieval. Ayende points out some problems with the dense, non-semantic
way some fields are stored in nuget, such as dependency versions and tags. He uses these as examples that illustrate
how RavenDB can handle nested collections of strings and complex objects.

That's great and all, but if we're redesigning the schema, we've already changed the rules. Nuget.org is having some
trouble specifically because of a poorly designed schema that requires too many joins and some other complex hoop-jumping
to execute some frequently used queries.

Furthermore, regardless of how the data is stored behind the server, NuGet uses an OData API to expose its packages
to clients. If you change the schema, you have to do it in a way that remains compatible with that client API, or
the millions of installed instances suddenly stop working.

## Semantic Version Sort vs. Lexical Sort ##

One of the primary ways that packages are sorted in NuGet is by version. Unfortunately, the complex nature of
semantic versions, such as 0.9-alpha or 1.0.5, makes it impossible to sort them correctly using a basic lexical sort.
For example, 1.0-alpha should get sorted before 1.0, and 1.2 should get sorted before 1.10. With lexical sort
they do not.

This problem was pointed out in a comment on [Part IV](http://ayende.com/blog/158210/nuget-perf-part-ivndash-modeling-the-packages)
but left unresolved.

## Review ##

In using NuGet as an example for RavenDB, the following features are not addressed:

1. Providing a backwards-compatible OData endpoint
1. Sorting correctly by package version
1. Storing and retrieving package contents (only package metadata is addressed)
1. Adding new packages into the index
1. Keeping track of download counters (per-package and total)

Without addressing adding new packages and tracking downloads, the data store is effectively read-only,
meaning that caches never get invalidated and there are no writes to disk. Based on this criteria, is
it really fair to compare performance benchmarks between this sample and a real, production system?

## The Load Tests ##

In the next part of the series, the sample system is put under some load to see what kind of performance
RavenDB can provide when there are concurrent users. [The Tests](http://ayende.com/blog/158530/nuget-perf-the-final-part-ndash-load-testing-ndash-the-tests)
goes into some detail about the load testing plan.

Reviewing the load test, we have another obvious problem: only a handful of sample queries are being used.
When the number of search queries is low, say, under 10, the system under load can simply cache the results
from the previous time the query was executed and return that to the user. Since there isn't any variance
in which columns are used to sort, what page is retrieved, etc., the system under load doesn't have to think
very hard at all.

Combine the simplicity of the test plan with the fact that the system under test is not doing any writes at all,
and you might as well be slinging static html at that point. I guess it does validate, if nothing else, that
the caching built into RavenDB works.

One indicator that the load test isn't really hitting any critical thresholds is that Pages/Sec grows pretty
linearly as User Load ramps up. If the system hit such a threshold, one would expect Pages/Sec to peak at
a certain rate and (ideally) sustain that rate as concurrency continues to grow.

## Summary ##

To reiterate, I'm not trying to bash RavenDB, which I'm sure is an awesome product and probably really would
perform very effectively under load similar to that experienced by [nuget.org](http://nuget.org/). However,
casual observers may come away with unrealistic expectations after reading the performance results offered
up on [ayende.com](http://ayende.com).

## Why Do I Care ##

If Ayende had picked virtually any example other than NuGet packages to do a blog series on, I probably would have
read along and not thought much about any of it. But having worked on NuGet.Server + Lucene.Net.Linq for
several months earlier this year, the subject matter is near and dear to my heart.

In future posts I'll get into some more compare and contrast on RavenDB vs. [Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq)
and maybe do a load test of my own of our [custom NuGet.Server builds](https://github.com/themotleyfool/NuGet/downloads).

