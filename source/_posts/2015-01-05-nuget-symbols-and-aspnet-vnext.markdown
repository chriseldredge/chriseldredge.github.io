---
layout: post
title: "NuGet Symbols and Aspnet vNext"
date: 2015-01-05 14:52
comments: true
categories: 
---

I've been working on a new release of Klondike for quite a while now.
My last release was [published](https://github.com/themotleyfool/Klondike/releases)
on March 27th, 2014.

There's been quite a bit of work off and on over the months, but I've kept
postponing a new version to add Just One More Feature. Scope creep, anyone?

One remaining issue I'm now working through is dealing with normal NuGet packages,
NuGet symbol packages, and looking ahead at Microsoft.NET 5 (aka .NET vNext).

Ignoring vNext, the `nuget pack` command takes an optional `-symbols` argument
which asks it to build two separate zip files: one that contains the DLLs,
documentation, content, scripts and other artifacts that a project would consume
from your package, and another one that contains the DLLs, PDBs (debugging symbols),
and source code for your package.

The symbol packages are intended to be uploaded somewhere like [SymbolSource.org](http://www.symbolsource.org/)
where they will be processed for consumption by debuggers like Visual Studio and WinDbg.

Klondike allows both normal packages and symbol packages to be uploaded to a single
end point. This enables Klondike users to use NuGet for private libraries without
losing useful source level integration with Visual Studio.

On the surface, there is little to tell a symbol package from a normal one. Both packages
will have virtually identical manifests and metadata. The metadata will have a package
ID and version. Apart from the name of the package file (MyThing.1.0.nupkg vs MyThing.1.0.symbols.nupkg),
we have to fall back to heuristics to suss out one from the other.

This is what Klondike does: look at the files in the package and if there's at least one
file in the `src` directory, and at least one `pdb` file in the `lib` directory, then
the package is deemed to be a symbol package.

With vNext, the waters are muddied. Commands like `kpm pack` produce a single package
that contains consumable contents but also includes PDBs and source. This sounds
great. Two use cases in a single payload.

The trouble is in how we distinguish between a normal package and a symbol package.
Klondike needs to treat them differently so when a client asks to download a
package, it gets the full normal package and not just the symbols and source.

At present, Klondike wrongly concludes that a vNext package is a symbol package.
My heuristic is wrong when dealing with such packages.

## It's the file name stupid

My first thought was, it is obvious from a file name like `MyThing.1.0.symbols.nupkg`
that a package should be treated as symbols-only. When files are uploaded using
multipart/mime, a helpful Content-Disposition header should tell me the name of the
file being uploaded.

Well, as it turns out, regardless of what the file is named on disk, the `nuget push`
command always lies to the server and says it's uploading a file named `package`.

Push `MyThing.1.0.nupkg` and the server sees `package`. Push `MyThing.1.0.symbols.nupkg`
and the server sees `package`. Push a file that isn't a package and the server sees...
you get it.

## It's the endpoint stupid

The client knows if a package is symbols or not. Maybe the client should use a different
endpoint to upload symbol packages from normal ones. Perhaps trying to handle both package
types from the same URL was a bad decision in the first place.

Changing it now would break any automated scripts that currently push packages to Klondike.
Maybe there's not a ton of clients out there today, but it still rubs me the wrong way.

## It's something else?

Apart from requiring the client to use a different URL to push symbols, I'm at a loss of
how else to reliably tell the difference between symbols and normal packages. Since
the problem only affects vNext, I could add more complicated heuristics to see if
`aspnet50` or `aspnetcore50` are supported by the package. This does not sounds like
an approach that won't need constant attention as new frameworks and versions are
introduced.

## What do

It's a bummer that symbol packages don't have a simple flag in their manifests that proclaim
what they are. It's a bummer that `nuget push` uses a hard-coded file name no matter what
type of package is being uploaded. It's a bummer that vNext appears to be changing
the rules of what a normal package may contain.

Unless I've overlooked something, the right choice seems to be to require clients
to push symbol packages to an alternate URL.


