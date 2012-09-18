---
layout: post
title: "How The Motley Fool Uses Octopus Deploy"
date: 2012-09-06 11:23
comments: true
categories: 
---

I've mentioned a few times that my team has been using [Octopus Deploy](http://octopusdeploy.com/) in a few of my posts.
Now I'll describe in more detail some of the ways we've integrated with Octopus that may help others.

## The Before Times ##

Octopus started a beta phase in late 2011, so what were we doing before? Since I started working in an asp.net shop
in 2006, I've been surprised by the lack of robust deployment tools for the ecosystem.

The Web Deployment Projects system provided by Microsoft has always seemed like a non-starter for me. When you
have mature processes like version control, continuous integration, automated testing, and quality assurance,
how does it make sense to hand the keys over to a developer running Visual Studio to click "Deploy"
on their desktop? What guarantee do you have that the code has been checked in, that it builds, that
the tests pass, that QA signed off?

Even if you only give trusted users permission to deploy, I've never understood how it makes sense that you
would need Visual Studio to deploy your projects. I mean, Visual Studio is for development. It isn't even
needed by a build server to compile your code. But now QA (or whoever is allowed to perform deployments) needs
to have Visual Studio installed somewhere and checkout the code and build it themselves? So much for automation.
So much for predictability. So much for repeatability.

So it was with a lack of viable alternatives that I started writing a deployment tool in 2007. The tool was
named Bazooka (because it "shoots" software onto servers), and we used it for 5 years. We never released it as
open source because it made several assumptions particular to us and we didn't take the time to clean it up.

Bazooka was a web application that watched a directory for release candidates to show up as they were
prepared by our build server. Each release candidate contained a deployment descriptor that had some
meta data and a list of components that could be deployed to servers that matched certain roles like
WebServer or AppServer.

Bazooka used xml configuration for everything. The deployment descriptor was xml, the server and environment
configuration was xml, the permissions were xml, and the log files that recorded what had been deployed where
were also xml. This made Bazooka pretty slow as deployment logs piled up, but it beat the heck out of
doing anything more heavy-weight with sql.

Bazooka was, in reality, just a web interface to MSBuild. When you went through the deployment wizard,
Bazooka simply executed MSBuild in the directory specified by each component with some properties saying
which server to deploy to, which target to execute, what the environment was, etc.

We could have easily kept improving Bazooka, but we had actual code to write for our business, so Bazooka
had some weaknesses that we never really addressed. For example, managing servers and environments was
done by hand-editing the xml files. We didn't build a UI for that. Also, it was pretty tough to answer
simple questions like, "What was the most recent deployment of my project to the staging environment?"

But it worked as a fine stand-in for 5 years until someone built something better.

## Octopus Trial ##

It was fun playing with Octopus during the beta period and seeing another approach to a web based
deployment tool. In the beginning there were plenty of things Bazooka had that Octopus didn't (yet),
but development has been pretty rapid with Octopus delivering frequent releases.

The one time where we asked if using Octopus was right was when Paul Stovell, the creator/developer
of Octopus Deploy, took a few months to rewrite the persistence layer, converting it from Entity
Framework to RavenDB. It was the right decision to make and we're happy in the long run, but we
found ourselves in a lurch waiting for some important features and bugfixes. We came out fine on
the other side though.

We also found that using a local file-based NuGet feed with Octopus doesn't scale very well, and
switching to NuGet Server provided no benefit either. This inspired us to create a
[custom fork](https://github.com/themotleyfool/NuGet) of NuGet Server that uses
[Lucene.Net](http://incubator.apache.org/lucene.net/) and
[Lucene.Net.Linq](https://github.com/themotleyfool/Lucene.Net.Linq) to provide a scalable, lightning fast internal feed for Octopus.

## The Switch ##

As we integrated some pilot projects with Octopus, we slowly stopped using Bazooka and
eventually turned off Bazooka integration. Some quick stats of our Octopus configuration today:

 * 48 projects
 * 18,080 release candidates
 * 1,405 deployments

Octopus has done a decent job managing our high demands. We have experienced some slow page
loads here and there, but Paul has been very responsive about troubleshooting and optimizing
these.

## Reusing Deployment Scripts ##

Since we already had a highly automated deployment system, we wanted to preserve our exising
capabilities while finding ways to improve the system.

One problem that Octopus doesn't solve for you is how to share deployment scripts across
projects. By default, Octopus will execute deployment scripts contained in each project,
but there isn't a built-in or standard way to reuse common functionality.

One of the first projects we set up in Octopus we named OctopusScripts. This project
consists of a collection of PowerShell modules that we want to be available everywhere.
When we deploy the project, the deployment scripts install the modules into a
standard location where PowerShell will probe for them. Then from other projects,
we can simply start a script with:

```
Import-Module SmokeTest
```

## Reuse Moar ##

Moving most of our scripts into PowerShell modules was working great, but we started
to notice that our Deploy.ps1 scripts still looked awfully repetitious.

All of our web projects follow the same basic deployment recipe:

1. PreDeploy
    1. Disable machine in load balancer
    1. Create IIS site and app pool definitions if missing
    1. Update site host-header bindings as needed
1. Let Octopus update the document root to point to the new application
1. PostDeploy
    1. Execute smoke tests against the server and abort if any URLs return a non-200 response
	1. Enable machine in load balancer
	
Of course we have different configurations and different URLs to use for smoke tests
for each project. We ended up creating a configuration based, modular script collection
so that each project simply needs to include a stub:

```
Import-Module Fool-Octopus
Invoke-OctopusDeploymentTasks
```

The Invoke-OctopusDeploymentTasks function looks at whatever variable and configuration
are present and figure out which steps to execute. The same scripts are used for Windows
Service type projects and others too, and they figure out based on conventions if they
need to run web steps, create services, etc.

If the stub is missing from a project (because why duplicate the stub?) our build scripts
automatically insert a stub Deploy.ps1, PreDeploy.ps1 and PostDeploy.ps1.

We think we're about as DRY as we can get with regards to our deployment scripts.

## Ad-hoc PowerShell ##

One thing we'd like to see that hasn't made it into Octopus yet is the concept of ad-hoc
powershell scripts. Basically, we want to be able to run some arbitrary scripts during a
deployment, only once. It doesn't need to run once on each machine being deployed to, just
once and then done. There's a [story card](https://trello.com/card/new-deployment-step-ad-hoc-powershell/4e907de70880ba000079b75c/8)
on Paul's Trello board that we're looking forward to. In the mean time, we've been emulating
this behavior by deploying a small package to a dummy server and letting the script run there.

We mostly want this feature to simplify tasks such as sending email/other notifications when
deployments are beginning or completed. It might also be useful for a green/blue style
deployment model where the load balancer needs to toggle just once after the servers
have been updated.

Without the Ad-hoc feature, one of the stumbling points we run into is sometimes forgetting
to check the "Force redeployment" checkbox that Octopus leaves unchecked. When we forget,
some steps get skipped leading to confusing results.

## Looking Ahead ##

Because of the level of automation we integrated with Octopus, our business is able to
deploy software more frequently and more reliably than ever before. In the coming
months and years, we look forward to seeing improvements in Octopus features that will
help us with cloud deployments to AWS or Azure. Octopus has definitely filled a gap
in our deployment capabilities, allowing us to deliver value to our business quickly,
iteratively and predictably.

