---
layout: post
title: MSBuild, Visual Studio & NuGet Hackery
---

My team at work recently switched to using NuGet. We have a fair amount of shared code that used to be in a
monolithic code base with all the problems that came with it. Slow builds, no concept of versions, tight coupling, etc.
To address these issues we started migrating to a new layout where we split Framework Projects out into seperate
repositories and publish NuGet packages for those projects. Then our Application Projects use NuGet to depend on
the Framework Projects and we get better control of when dependencies are upgraded, faster builds and everyone is happy.

One drawback to this migration is that we no longer have all projects in one Visual Studio solution, making it hard to
work on writing a new feature or fixing a bug in a Framework Project and testing that change in an Application Project.
Where we used to be able to do this seemlessly, we now have to jump through some hoops:

1. Change Framework Project
1. Build Framework Project
1. Manually copy the dll to the Application Project
1. Change Application Project to use new code (if necessary)
1. Build Application Project

Once the code changes are deemed acceptable, additional work is necessary to commit/push changes to other developers:

1. Commit/Push Framework Project
1. Wait for build server to publish a NuGet package
1. Use NuGet to update Framwork project dependency in Application Project
1. Commit/Push Application Project

This multistep process obviously slows down development significantly. My mission is to find a way to cut out some
of these manual steps so our developers can get back to writing and refactoring code instead of wrestling with tools.

__ SlimJim __

A colleague of mine, Aaron Togerson, started a project a while back called SlimJim.
[SlimJim](https://github.com/AaronTorgerson/SlimJim/commits/master) is a Visual Studio Solution Generator.
I've always liked the phrase "Solution Generator" despite the fact that Visual Studio solution files have
little to do with solving actual problems.

Anyway, SlimJim is a command-line tool that analyzes c# projects and generates a <code>.sln</code> file
that includes a target project (such as a Framework Project) and all projects that depend on it. This
tool enables us to quickly generate Solutions that allow us to have Framework Projects and Application
projects open in a single instance of Visual Studio. Since SlimJim generates these files pretty quickly,
we never check them into source control.

Using SlimJim in conjunction with ReSharper we gain the ability to navigate through the code and apply
refactoring tools to rename types and methods, change method signatures, introduce parameters, and all
the other fun stuff.

__ The Pitfall __

It's great that ReSharper knows how to do this, but the problem is when you hit F6 or click Build Solution,
and you get a bunch of compilation errors. What's going on? First, NuGet injects assembly references into
the Application Project instead of injecting project references. Visual Studio uses project references to determine
the build order. So because there is no project reference from Application Project pointing to Framework Project,
the order in which the projects are built will be undefined. The second problem is that even if the build order
could be corrected, there is nothing in place that copies the build output from Framework Project over to
where Application Project will find it.

We could solve both problems by converting the c# project files for Application Projects to replace
&lt;Reference/&gt; items with &lt;ProjectReference/&gt; items. This is how Visual Studio decides what
order projects are built in and how outputs from one project get copied to another.

The problem is that we don't want to accidentally commit changes like this. We only want them to apply
to local development. Accidentally checking such changes in will certainly break the build and take us
back to the bad old days of slow builds.

__ Attempt 1 __

To avoid changing c# project files, the first approach I took was to try to do some MSBuild magic and
hope Visual Studio would get in on the game. MSBuild provides some hooks that enable us to manipulate
Item Groups dynamically. First, the [InitialTargets](http://msdn.microsoft.com/en-us/library/bcxfsh87.aspx)
attribute can be used to execute some targets before the requested targets are executed. Second, we
can do some analysis of the solution file being built and the project references to try to remove
&lt;Reference/&gt; items and add &lt;ProjectReference/&gt; items dynamically.

Note: this example uses tasks from the [MSBuild Community Tasks](https://github.com/loresoft/msbuildtasks) project.

{% gist 3177148 %}

This implementation works great in MSBuild on the command line when invoked on a c# project.
If you specify a SolutionPath and BuildingInsideVisualStudio,
MSBuild will faithfully build the Framework Projects before building the Application Project.

However, it doesn't work in Visual Studio. Evidently when Visual Studio loads a solution, it
either doesn't execute InitialTargets or it uses some other mechanism to extract &lt;ProjectReference/&gt;
items from each project.

__ Attempt 2 __

If we can't dynamically tell Visual Studio about project references, I thought maybe we could just build
the projects in the right order ourselves. MSBuild already does this when building outside Visual Studio
provided the BuildProjectReferences property is not false.

So I modified my MSBuild script to try to do that:

{% gist 3177381 %}

Once again, this works great in MSBuild outside of Visual Studio. When you load up the solution in VS
and hit build, the Output pane says encouraging things like "MyApp is building C:\Projects\MyFramework\src\MyFramework\MyFramework.csproj".

Except it doesn't. I couldn't figure out why, but somehow Visual Studio seems to know that we're trying to trick it, and it refuses
to actually build the project. It pretends to, but it doesn't. I thought this might have to do with In-Process Compilers and
other optimizations described in the [Visual Studio MSBuild Integration](http://msdn.microsoft.com/en-us/library/ms171468.aspx) article,
but attempting to disable <code>UseHostCompilerIfAvailable</code> has no effect.

I gave up on this strategy becuase even if it helped to build projects in the right order, it doesn't copy build outputs from one
project to the other.

__ Attempt 3 __

Once again, Visual Studio proved to be too enterprisey for us to customize. The only proof of concept I was able to make work
was the dumb one: converting c# projects and risking that the converted versions get checked in. We can solve the accidental commit
issue with a pre-commit hook, so maybe it isn't all bad as long as we can seemlessly convert back and forth.

Since we already have a tool that analyzes c# project references, it makes sense to extend SlimJim to mangle c# projects while analyzing them.

My [fork](https://github.com/chriseldredge/SlimJim) of SlimJim now includes two new switches: <code>--convert</code> and <code>--unconvert</code>.

Suppose SlimJim finds two projects, MyApp and MyFramework, and MyApp.csproj looks like this:

``` xml MyApp.csproj
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup>
    <Reference Include="MyFramework, Version=1.0.0.0, Culture=neutral, PublicKeyToken=3c369c070579152a, processorArchitecture=MSIL">
      <HintPath>..\packages\MyFramework\lib\net40\MyFramework.dll</HintPath>
    </Reference>
  </ItemGroup>
</Project>
```

Using <code>--convert</code>, SlimJim will change MyApp.csproj to look like this:

``` xml MyApp.csproj after SlimJim
<Project ToolsVersion="4.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <ItemGroup>
    <ProjectReference Include="C:\projects\MyFramework\src\MyFramework\MyFramework.csproj">
      <Project>{673A97DA-4302-48C9-BFF0-A57FD7DBA93F}</Project>
      <Name>MyFramework</Name>
      <SlimJimReplacedReference>
        <Reference Include="MyFramework, Version=1.0.0.0, Culture=neutral, PublicKeyToken=3c369c070579152a, processorArchitecture=MSIL">
          <HintPath>..\packages\MyFramework\lib\net40\MyFramework.dll</HintPath>
        </Reference>
      </SlimJimReplacedReference>
    </ProjectReference>
  </ItemGroup>
</Project>
```

And using <code>--uncovert</code> will put it back. The original assembly reference is preserved in the markup, moved out of the
way so MSBuild and Visual Studio will ignore it, but making it easy for us to restore it without losing metadata like HintPath
and SpecificVersion.

As long as we remember to <code>--unconvert</code> before committing, this new featuer should provide a great benefit.
