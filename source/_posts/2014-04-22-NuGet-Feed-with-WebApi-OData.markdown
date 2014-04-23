---
layout: post
title: "NuGet Feed with WebApi OData"
date: 2014-04-22 10:25
comments: true
categories: nuget, odata
---

NuGet uses OData to provide package data to its clients. In
[previous posts](/blog/2012/03/29/Getting-Started-With-Relinq/)
I've written about implementing Lucene.Net.Linq and the difficulty
of implementing `IQueryable<T>` and leaky abstractions and such.

The NuGet core team has more or less acknowledged that OData is overkill
for the types of operations the client needs (list, search, find updates),
and they're even [planning](https://github.com/NuGet/NuGetGallery/issues/595)
on migrating away from OData in v3 of their HTTP api.

Unfortunately (or fortunately?) NuGet has achieved wide adoption using
the existing api, so it will take years for everyone to update their clients
once this new api has shipped.

In the mean time, we're stuck with OData.

To make the most of the situation, I wanted to try out the new WebApi
integrated OData packages. These packages allow us to use the same base classes
and infrastructure as regular WebApi controllers. They also enable us
to build and deploy self-hosted applications to decouple us from IIS.

WebApi OData has seen lots of features get implemented and adds supports
for newer protocol versions of OData. However, there are a few things that
the NuGet clients require that are not built in. This series will
go over configuring WebApi OData to create a NuGet compatible package feed.

## Required Features

OData is a huge standard and NuGet uses only a subset of capabilities.
Here's a list of what we'll need to support:

* Entity Sets
* Composite Keys
* Actions
* Default Streams
* $count meta-action
* $filter
* $orderby
* Paging with $top and $skip

Most of these capabilities are built into WebApi OData.
The ones that aren't will be covered in the following articles.

{% include custom/nuget_webapi_odata_series_index.html %}
