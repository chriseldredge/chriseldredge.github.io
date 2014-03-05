---
layout: post
title: "The newer new NuGet Package Restore"
date: 2014-01-29 13:01
comments: true
categories: 
---

When NuGet 2.7 was released, the topic of how to manage packages in your source
repository was [revisited][1] and the waters were muddied.

The new guidance is to not use the "Enable NuGet Package Restore" option in
Visual Studio, but instead to let it happen automatically.

Unfortunately, many developers will probably miss this guidance if they're already
used to restoring packages as part of their workflow.

The option offered when right-clicking the Solution in Visual Studio gives no
indication that the option is deprecated, and the text implies that if you don't
activate the option, packge restore will not be enabled.

These are minor annoyances that will hopefully be addressed as more users
adopt NuGet 2.7 and later clients.

---

One thing the guidance fails to address completely is how an automated build
system, like a Continuous Integration server, should approach restoring packages.

The section that covers this topic explains how packages can be restored using
the NuGet comand line client prior to building the solution, but where are
you supposed to get nuget.exe from?

There are 3 approaches you can take to address this:

1. Assume the build server is configured to have a recent nuget.exe on the PATH
1. Check nuget.exe into your source repository
1. Fetch the latest nuget.exe from nuget.org on the fly.

The now obsolete "Enable NuGet Package Restore" option did a mix of (2) and (3).
It would look for nuget.exe in the .nuget folder, and if it isn't found and some
options are enabled, it will download it from the web.

How can we continue using this feature while moving towards the new One True Way
of Package Restore?

We can do it by co-opting parts of [NuGet.targets][2] in our own build script.

First, layout:

```
project/
project/IntegratedBuild.proj
project/NuGet.targets
project/source/
project/source/Foo.sln
```

## IntegratedBuild.proj

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="12.0" DefaultTargets="Build" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <SolutionDir>$(MSBuildProjectDirectory)\source\</SolutionDir>
  </PropertyGroup>

  <ItemGroup>
    <Solution Include="$(SolutionDir)*.sln"/>
  </ItemGroup>

  <Target Name="Build" DependsOnTargets="RestoreSolutionPackages;BuildSolution"/>

  <Target Name="RestoreSolutionPackages" DependsOnTargets="DownloadNuGetCommandLineClient">
    <Exec Command="$(NuGetCommand) restore $(SolutionDir) -NonInteractive -Source http://www.nuget.org/api/v2/"/>
  </Target>

  <Target Name="BuildSolution">
    <MSBuild Projects="@(Solution)"/>
  </Target>

  <Import Project="NuGet.targets"/>
</Project>
```

## NuGet.targets

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <NuGetToolsPath>$(MSBuildThisFileDirectory)</NuGetToolsPath>
  </PropertyGroup>

  <PropertyGroup>
    <NuGetExePath Condition=" '$(NuGetExePath)' == '' ">$(NuGetToolsPath)\NuGet.exe</NuGetExePath>

    <NuGetCommand Condition=" '$(OS)' == 'Windows_NT'">"$(NuGetExePath)"</NuGetCommand>
    <NuGetCommand Condition=" '$(OS)' != 'Windows_NT' ">mono --runtime=v4.0.30319 $(NuGetExePath)</NuGetCommand>
  </PropertyGroup>

  <Target Name="DownloadNuGetCommandLineClient">
    <DownloadNuGet OutputFilename="$(NuGetExePath)" Condition=" !Exists('$(NuGetExePath)')" />
  </Target>

  <UsingTask TaskName="DownloadNuGet" TaskFactory="CodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.v4.0.dll">
    <ParameterGroup>
      <OutputFilename ParameterType="System.String" Required="true" />
    </ParameterGroup>
    <Task>
      <Reference Include="System.Core" />
      <Using Namespace="System" />
      <Using Namespace="System.IO" />
      <Using Namespace="System.Net" />
      <Using Namespace="Microsoft.Build.Framework" />
      <Using Namespace="Microsoft.Build.Utilities" />
      <Code Type="Fragment" Language="cs">
          <![CDATA[
          try {
              OutputFilename = Path.GetFullPath(OutputFilename);

              Log.LogMessage("Downloading latest version of NuGet.exe...");
              WebClient webClient = new WebClient();
              webClient.DownloadFile("https://www.nuget.org/nuget.exe", OutputFilename);

              return true;
          }
          catch (Exception ex) {
              Log.LogErrorFromException(ex);
              return false;
          }
      ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>
```

---

Make sure to add NuGet.exe to your .gitignore or equivalent scm configuration.

It's a little unfortunate that this boilerplate has to be copied from one project to another.
However, since we're bootstrapping NuGet we have a chicken and egg problem that prevents
the targets from being shared in a package.

I hope this helps developers get the most out of NuGet's latest approach to restoring
packages while avoiding needing to check NuGet.exe into source control and keep it up to
date.

This won't work with xbuild on Mono since it does not support CodeTaskFactory, but if
you are using mono you could add an alternate target that uses curl or wget to perform
the download. Assuming curl or wget are already on your PATH. Sigh.

---

A simpler approach if you know curl is on the system is to use a different scripting
language, that effectively does

```
curl -o nuget.exe https://www.nuget.org/nuget.exe
./nuget.exe restore source -NonInteractive
msbuild source\*.sln
```

But why use 3 lines of script when you can have a 100 lines of XML?

[1]: http://docs.nuget.org/docs/reference/package-restore
[2]: http://nuget.codeplex.com/SourceControl/latest#src/Build/NuGet.targets
