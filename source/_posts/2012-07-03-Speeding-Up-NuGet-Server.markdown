---
layout: post
title: Speeding Up NuGet.Server
comments: true
categories:
---

_(Get the source code at [https://github.com/themotleyfool/Klondike](https://github.com/themotleyfool/Klondike))_

Last time I wrote about creating a LINQ provider for Lucene.Net, and today I'll talk about integrating that provider
with NuGet. The existing server part of the NuGet codebase is a drop-in replacement for using local file-system based
feeds. I wanted to try to preserve that turnkey advantage but improve the performance of various queries.

In order to make sure that my improvements were up to snuff, I set up a private mirror of all packages on [nuget.org](http://nuget.org/),
which turned out to be 44,193 packages at the time, for a total size of over 20 gigs.

If you try hitting ~/api/v2/Packages on stock NuGet.Server, you'll find that your request just spins and spins. And spins. In fact
it took so long that I gave up waiting for the application to initialize. In the background, the server is finding all <code>*.nupkg</code>
files in ~/Packages and calculating a hash of the contents. Needless to say, it can take a while to run a checksum algorithm on 20gb
of data.

Switching over to my custom lucene branch, the first time the site is started, it scans the Packages folder and finds all packages
that haven't been indexed by Lucene. The site homepage helpfully tells you the current status, such as "Indexing 2113 of 44193 new packages."
An ajax timer refreshes the info every few seconds so progress can be easily tracked.

The packages don't begin to appear in the feed until they've all been indexed. So this isn't much better than stock NuGet.Server.

## Incremental Indexing ##

The real improvements are appreciated after the initial index is built.

    [celdredge@localhost]$ appcmd recycle apppool nuget
	"nuget" successfully recycled

    [celdredge@localhost]$ time wget -O /dev/null http://localhost/api/v2/Packages

	(snip)

    real    0m3.230s
    user    0m0.062s
    sys     0m0.125s

This means that you don't have to worry much about IIS shutting down the application during idle times. The index gets loaded and ready to go
in a matter of seconds. Vast improvement over stock NuGet.Server.

While that happens, a background thread scans the Packages folder to see what might have changed while the application was stopped. New, modified
and deleted packages are synchronized with the Lucene index. The sycnhronization process takes about 25 seconds to scan 44,193 package files
split into 6,180 folders and calculate the differences with the Lucene index. That's pretty fast.

After the application finishes this initial scan, a FileSystemWatcher monitors the Packages folder to synchronize any changes in real time.
This allows the index to stay in sync when new packages appear, even if they are copied into the folder instead of using <code>nuget push</code>.

## Superfast Search ##

All sorts of complex queries are possible, and they execute in very reasonable time. I used [LINQPad](http://www.linqpad.net/) to
construct various test queries, like this one that finds packages whose id contain lucene but do not start with lucene:

    from p in Packages
    where p.Id.Contains("Lucene")
    where !p.Id.StartsWith("Lucene")
    where p.IsLatestVersion
    orderby p.Id descending
    select p

	Query successful (00:00.136)

136ms is pretty respectable, IMO.

Another advantage to using Lucene is how queries are analyzed. Term queries will match various word forms, so a query like <tt>build</tt> will
match packages that use any words like build, builds, building, built, etc. It is also possible to search for phrase queries, such as
<tt>"glue them back together"</tt>. That query matches only one package that contains the exact phrase, whereas on nuget.org you'll get
all kinds of results.

## Other Features ##

The [Tab Completion API Endpoints](https://github.com/NuGet/NuGetGallery/wiki/Tab-Completion-API-Endpoints) introduced in NuGet 2.0 have
been implemented, bringing fast results to users of the Package Manager Console.

## Conclusion ##

It has taken a substantial amount of time and effort to implement Lucene.Net.Linq and integrate it with NuGet.Server, but the results
have proven to be worth the investment.

Lucene.Net.Linq has become a fairly mature, though still nascent, project now available on [nuget.org](http://nuget.org/packages/Lucene.Net.Linq). There are a few other
OSS projects that attempt to do what it does, but I think it is already one of the best.

Binaries of NuGet.Server + Lucene can be downloaded from [https://github.com/themotleyfool/Klondike/releases](https://github.com/themotleyfool/Klondike/releases).
