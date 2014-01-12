---
layout: post
title: "Cross-NuGet Development With Slimjim"
date: 2014-01-12 16:00
comments: true
categories: 
---

Last week Phil Haack observed:

{% tweet https://twitter.com/haacked/status/421160899856969728 %}

I think Phil is trying to solve the same problem lots of .NET developers
run into. Let's say you have a web project that is using the wonderful
[Lucene.Net.Linq package](https://github.com/themotleyfool/Lucene.Net.Linq/)
and you want to fix a bug or add a feature to the library, and test
it in your web project without needing to manually copy DLLs around
from one solution to another.

This is a workflow I use constantly, and the tool I use to minimize
friction during development is [Slimjim](https://github.com/themotleyfool/Slimjim).

Slimjim, in a nut, is a command line tool that generates Visual Studio solution
files. The killer feature is that it will also modify csproj files
to replace assembly references with project references. This way
you can have a solution with all the projects you want to work with,
and when you hit &lt;ctrl>+&lt;shift>+B to build the solution, everything
builds in the right order and your top level project gets the
newly compiled code without you needing to do any extra work.

## An Example

Most of my open source work recently has been on [Klondike](https://github.com/themotleyfool/Klondike),
a fast NuGet package server you can host on your private network.

This project has actually exploded into lots of sub-projects,
each of which correlates to one or more NuGet packages available
on nuget.org.

### Getting the Source

First, we need to clone all of the repositories that have code
we want to work on.

    mkdir workspace
    cd workspace
    git clone https://github.com/themotleyfool/Klondike
    git clone https://github.com/themotleyfool/NuGet.Lucene
    git clone https://github.com/themotleyfool/Lucene.Net.Linq
    git clone https://github.com/themotleyfool/AspNet.WebApi.HtmlMicrodataFormatter

I said there were a lot of projects right? For fun let's pull in some 3rd party
open source code too. That way we can get debug builds and browse the code
to better understand it.

	git clone https://git01.codeplex.com/nuget
	svn checkout https://svn.apache.org/repos/asf/lucene.net/tags/Lucene.Net_3_0_3_RC2_final/ Lucene.Net
    
And note in some cases you may want to checkout a tag or branch other
than master. For nuget:

	cd nuget
	git checkout Release-2.7.2
	cd ..

### Getting Slimjim

Slimjim is on nuget.org, so you can install it locally by doing:

	nuget install slimjim

And now we have our utility in ./Slimjim.1.0.3/tools/Slimjim.exe

### Generating a Solution

Running Slimjim without any flags will create a Solution that includes
all csproj projects found under the current directory.

There may be projects you don't care about that you don't want
cluttering your workspace. For example, Nuget has lots of projects and
I only depend on Nuget.Core.

Since I'm working on Klondike, I'll tell Slimjim to target that project
and include anything it depends on, recursively.

    ./Slimjim.1.0.3/tools/Slimjim.exe -t Klondike -a -c

The `-a` switch says to include efferent dependencies, and the `-c` switch
says to modify the csproj files to convert assembly references to project
references.

If we look at one of the csproj files, we can see an example of this:

```diff
diff --git a/source/Klondike/Klondike.csproj b/source/Klondike/Klondike.csproj
index 049d8eb..df61e4b 100644
--- a/source/Klondike/Klondike.csproj
+++ b/source/Klondike/Klondike.csproj
@@ -45,10 +45,16 @@
       <SpecificVersion>False</SpecificVersion>
       <HintPath>..\packages\Antlr.3.5.0.2\lib\Antlr3.Runtime.dll</HintPath>
     </Reference>
-    <Reference Include="AspNet.WebApi.HtmlMicrodataFormatter, Version=2.0.0.0, Culture=neutral, processorArchitecture=MSIL">
-      <SpecificVersion>False</SpecificVersion>
-      <HintPath>..\packages\AspNet.WebApi.HtmlMicrodataFormatter.2.0.0\lib\net40\AspNet.WebApi.HtmlMicrodataFormatter.dll</HintPath>
-    </Reference>
+    <ProjectReference Include="C:\Projects\workspace\AspNet.WebApi.HtmlMicrodataFormatter\source\AspNet.WebApi.HtmlMicrodataFormatter\AspNet.WebApi.HtmlMicrodataFormatt+      <Project>{794ACAF9-D25C-4571-B801-9FC2CF86C1B4}</Project>
+      <Name>AspNet.WebApi.HtmlMicrodataFormatter</Name>
+      <SlimJimReplacedReference>
+        <Reference Include="AspNet.WebApi.HtmlMicrodataFormatter, Version=2.0.0.0, Culture=neutral, processorArchitecture=MSIL">
+          <SpecificVersion>False</SpecificVersion>
+          <HintPath>..\packages\AspNet.WebApi.HtmlMicrodataFormatter.2.0.0\lib\net40\AspNet.WebApi.HtmlMicrodataFormatter.dll</HintPath>
+        </Reference>
+      </SlimJimReplacedReference>
+    </ProjectReference>
     <Reference Include="Common.Logging, Version=2.1.2.0, Culture=neutral, PublicKeyToken=af08829b84f0328e, processorArchitecture=MSIL">
```

Remember not to commit this or you'll have a bad time!

### Fixing some NuGet things

This is where we'll hit some friction. I'd like to improve Slimjim to handle
some of these problems, but the good news is you only have to do this once.

If any of your projects are using NuGet's package restore feature, your newly
generated solution probably won't build. This is because NuGet expects to have
some resources available relative to the `$(SolutionDir)` property, and
our Solution file is in a directory that doesn't have these tools.

I usually copy them from one of the projects:

    cp -R Klondike/source/.nuget .

Additionally, assembly hint paths are hard-coded in the csproj files, usually like:

    <HintPath>..\packages\SomeLib.1.0\lib\net40\SomeLib.dll</HintPath>

However, when you build from the solution file in workspace, NuGet will restore packages
relative to this directory, so the csproj files won't be able to see them.

The simple fix is to build each project individually so it will restore packages
to the right place. After each individual repo's solution builds, we'll be able
to build from our Slimjim solution.

The more complex solution is to use NTFS junctions to make sure each repo is
using the same package directory.

	mkdir packages
    cd Lucene.Net.Linq/source
    mklink /j packages ..\..\packages

### Doing actual work

Now we have a Solution with the projects we want to work on and it should
build successfully from the command line using msbuild.exe.

We can open the Solution in Visual Studio, write code, build, test with IIS
or IIS Express, run unit tests, write more unit tests, and all the other good
stuff we like doing.

### Releasing

Once we've made changes we're happy with, we have to undo the changes Slimjim
made to our csproj files before we can commit:

    ./Slimjim.1.0.3/tools/Slimjim.exe -t Klondike -a -u

The `-u` switch is the correlary to `-c`, restoring assembly references that
we had turned into project references before.

Now we can commit/push/pull request, push packages to nuget.org, then
update packages in the projects that need them.

### Looking ahead

This workflow is not perfect. It requires some manual setup and upkeep
whenever we want to pull changes and keep everything in sync. It introduces
a danger in that you might accidentally commit stuff to your csproj. If
you update any nuget packages while working in your generated Solution,
the hint paths will point to the wrong packages folder.

I'd like to explore how Slimjim can address some of these problems.
It would also be nice if nuget was a little more dynamic about hint paths.

In the mean time, Slimjim makes it possible to write features across
disparate projects without needing to push new binaries or manually
copy DLL files around.

### Git submodules

Coming back to what @haacked wanted to do, it would be possible to
treat your workspace folder as a new git repo and include each repo
within as a submodule. This could reduce some friction for teams
that want to work using a shared pattern.