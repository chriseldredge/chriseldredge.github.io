---
layout: post
title: Getting Started with Re-linq
---

## The IQueryable Conundrum ##

So, you have a datasource you can access from .net and you thought, hey wouldn't it be
cool if I could wire this up with LINQ?

Maybe you've read about how [IQueryable is Tight Coupling](http://blog.ploeh.dk/2012/03/26/IQueryableIsTightCoupling.aspx),
but you figure that you're connecting up a low-level datasource and you'll probably encapsulate the messy IQueryable stuff
in your domain. So you forge ahead.

You create a class that implements IQueryable, maybe using
[Walkthrough: Creating an IQueryable LINQ Provider](http://msdn.microsoft.com/en-us/library/bb546158.aspx)
as a starting point.

Now you're left with an Expression object and you get to figure out what to do with it. Sure, you
can use reflection like the MSDN example demonstrates, but some lingering doubt enters in when you
realize how complex even the most basic statements will be to parse.

Then maybe you find a new resource: [LINQ: Building an IQueryable provider series](http://blogs.msdn.com/b/mattwar/archive/2008/11/18/linq-links.aspx)
and you think, hey, only 17 massive posts to trudge through.

Ok, so this is no work for the weary. No wonder we don't have scads of LINQ providers floating around on codeplex.com and codeproject.com.

## Enter re-linq ##

[re-linq](http://relinq.codeplex.com/) is an open source project that attempts to ease the burden of implementing IQueryable.

It seems like a great project that should help you write less code, but there's just one problem: re-linq has zero documentation.

Their project page links to the tantalizingly friendly example [re-linq|ishing the Pain: Using re-linq to Implement a Powerful LINQ Provider on the Example of NHibernate](http://www.codeproject.com/Articles/42059/re-linq-ishing-the-Pain-Using-re-linq-to-Implement),
but after you download the code you realize soon enough that the project is outdated and doesn't even compile against re-linq 1.13.

Alas, you have one demo project to go on as supplied in the re-linq source bundle: Linq.LinqToSqlAppender. Needlesstosay,
this project is arguably more complex than the actual support library it builds on. For someone trying to get started, looking
at a provider that generates SQL queries is probably a little too complex of a starting point.

## The Bare Bones ##

Submitted for your entertainment, here lies the recipe for the most basic shell you can use to build upon re-linq.

First, create a new c# project.  Then using Nuget or going on your own, grab a copy of Remotion.Linq
(In the Package Manager Coneole, type <tt>Install-Package Remotion.Linq</tt>)

One you have your project ready, here's your code:

<script src="https://gist.github.com/2246429.js"> </script>

This implementation completely ignores the "where" clause, "order" clause, and any other clause you can
name besides "select".  It just returns the entire collection of items in the data source.

Up next, translating the where clause.
