---
layout: post
title: "Fun with NuGet Web Api"
date: 2013-02-25 10:57
comments: true
categories: 
---

NuGet is a package management system for .net. Like most things these days, NuGet uses HTTP
for client/server communication to enable search, download, upload and other operations.

Since I've been working on writing a NuGet web application, I have had the opportunity to
become intimately familiar with the web api and its wrinkles.

I was going to title this post "Fun with the NuGet REST Api", but then I had to change it
because what NuGet does is pretty far from RESTful. The documentation for
[Tab Completion Endpoints](https://github.com/NuGet/NuGetGallery/wiki/Tab-Completion-API-Endpoints)
openly acknowledges this, stating, "We currently don't look at the Accept header or do lots of other
proper HTTP API stuff". Caveat emptor.

There have been plenty of well written posts about what REST is and why most things that
speak json or xml over HTTP are not Restful.

1. [Clarifying REST](http://kellabyte.com/2011/09/04/clarifying-rest/) from @kellabyte
1. [Representational State Transfer (REST)](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm) Roy Fielding's doctoral thesis
1. [REST APIs must be hypertext-driven](http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)

WCF Data Services and REST
--------------------------

NuGet has used WCF Data Services (OData) since inception to do the heavy lifting.
OData kind of sort of does a decent job of adhering to RESTful constraints and
using hyperlinks that give hints to clients about collections, entities and
functions that can be invoked on them.

Query the API Root:

    curl http://nuget.org/api/v2/

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<service xml:base="http://nuget.org/api/v2/" xmlns:atom="http://www.w3.org/2005/Atom" xmlns:app="http://www.w3.org/2007/app" xmlns="http://www.w3.org/2007/app">
  <workspace>
    <atom:title>Default</atom:title>
    <collection href="Packages">
      <atom:title>Packages</atom:title>
    </collection>
  </workspace>
</service>
```

You can see there's a collection of Packages, and you even get an href that tells you where it is.

Unfortunately it does not tell you about the magic $metadata RPC call that gets you more
information about the entities contained in the Packages collection, and what other functions
are available:

    curl 'http://nuget.org/api/v2/$metadata'

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<edmx:Edmx Version="1.0" xmlns:edmx="http://schemas.microsoft.com/ado/2007/06/edmx">
  <edmx:DataServices xmlns:m="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata" m:DataServiceVersion="2.0">
    <Schema Namespace="NuGetGallery" xmlns:d="http://schemas.microsoft.com/ado/2007/08/dataservices" xmlns:m="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata" xmlns="http://schemas.microsoft.com/ado/2006/04/edm">
      <EntityType Name="V2FeedPackage" m:HasStream="true">
        <Key>
          <PropertyRef Name="Id" />
          <PropertyRef Name="Version" />
        </Key>
        <Property Name="Id" Type="Edm.String" Nullable="false" m:FC_TargetPath="SyndicationTitle" m:FC_ContentKind="text" m:FC_KeepInContent="false" />
        <Property Name="Version" Type="Edm.String" Nullable="false" />
        <Property Name="Authors" Type="Edm.String" Nullable="true" m:FC_TargetPath="SyndicationAuthorName" m:FC_ContentKind="text" m:FC_KeepInContent="false" />
        <snip/>
      </EntityType>
      <EntityContainer Name="FeedContext_x0060_1" m:IsDefaultEntityContainer="true">
        <EntitySet Name="Packages" EntityType="NuGetGallery.V2FeedPackage" />
        <FunctionImport Name="Search" EntitySet="Packages" ReturnType="Collection(NuGetGallery.V2FeedPackage)" m:HttpMethod="GET">
          <Parameter Name="searchTerm" Type="Edm.String" Mode="In" />
          <Parameter Name="includePrerelease" Type="Edm.Boolean" Mode="In" />
        </FunctionImport>
        <FunctionImport Name="FindPackagesById" EntitySet="Packages" ReturnType="Collection(NuGetGallery.V2FeedPackage)" m:HttpMethod="GET">
          <Parameter Name="id" Type="Edm.String" Mode="In" />
        </FunctionImport>
      </EntityContainer>
    </Schema>
  </edmx:DataServices>
</edmx:Edmx>
```

This xml vomit says that in addition to the hard-coded, out-of-band RPC API that is OData, you
can also invoke the Search and FindPackagesById functions, how to invoke them
and what parameters they take.

At some point the NuGet developers must have gotten sick of using WCF Data Services, because the endpoints for
uploading a package, deleting a package, and the tab completion endpoints all are implemented
without integration with WCF Data Services. That means that these operations are not discoverable
by a client, so if the client wants to invoke them it has to know their locations.

The Evolution of a Public API
-----------------------------

When the client has no way of discovering available resources, that means the server can never
change the namespace of those resources, or risk breaking older clients.

SalesForce has at least 26 namespaced versions of some services which causes some problems
that @kellabyte discusses on a [twitter thread](https://twitter.com/kellabyte/status/276661580257701889).

NuGet in its current form uses /api/v2 for most HTTP API calls, and has managed to stay on v2 for a while
now. We've seen some new features get added without forcing everyone to move over to v3 yet.
The v1/v2 crutch should never have been introduced, but since it is we're stuck with it.

Dumb Servers and Dumber Clients
-------------------------------

Since the server won't tell the client how to invoke certain operations, the client
must make assumptions and NuGet does exactly this. Badly.

The tab completion endpoints are predictably dumb. If you deploy a NuGet feed at
http://example.com/ and point Visual Studio to it, the PowerShell console will
try to go to http://example.com/api/v2/package-ids. If you deploy the same feed at
http://example.com/nuget, and point Visual Studio to that, the PowerShell console will
still try to go to http://example.com/api/v2/package-ids. Note how the /nuget/ part of
the namespace vanished. This will generally result in a 404.

Try to push or delete a package and things get more confusing. If you do
"nuget push Foo.nupkg -source http://example.com/", instead of blindly appending
api/v2 to the URI, the client first does a GET request on the root URI.
If that GET request results in a 301/302 Redirect, the client will follow the redirect
and use that location to push the package. However, if the root URI returns some other
response code, the client reverts to dumbly hard-coding the path to api/v2/package.

But if you push to a URI that is not the root URI, the client listens to you
and pushes to whatever URI you told it.

What all of this means is that it is surprisingly hard to write a web application
that complies with all the weird shit that the NuGet client does. If you tell
Visual Studio that the package feed is at http://example.com/api/v2, then NuGet will
push to http://example.com/api/v2. But if you tell NuGet to publish to
http://example.com/ then it will publish to http://example.com/api/v2/package.

That means you need redundant routes to catch both cases.

Example Routes
--------------

For my application, here are the routes I came up with that seem to keep
the NuGet client happy, as long as you don't deploy it as a child application:

```c#
public static void MapApiRoutes(HttpRouteCollection routes)
{
    // Serves up HTML to browsers that accept text/html
    routes.MapHttpRoute(RouteNames.Home,
                        "",
                        new { controller = "Home" },
                        new { acceptHeader = new AcceptHtmlConstraint() });
 
    routes.MapHttpRoute(RouteNames.PackageDownload,
                        "api/v2/package/{id}/{version}/content",
                        new { controller = "Packages", action = "DownloadPackage" });
 
    routes.MapHttpRoute(RouteNames.PackageInfo,
                        "api/v2/package/{id}/{version}",
                        new {controller = "Packages", action = "GetPackageInfo", version = ""},
                        new { httpMethod = new HttpMethodConstraint(HttpMethod.Get) });
    
    routes.MapHttpRoute(RouteNames.PackageApi,
                        "api/v2/package/{id}/{version}",
                        new { controller = "Packages", id = "", version = "" },
                        new { httpMethod = new HttpMethodConstraint(HttpMethod.Put, HttpMethod.Post, HttpMethod.Delete) });
    
    routes.MapHttpRoute(RouteNames.TabCompletionPackageIds,
                        "api/v2/package-ids",
                        new { controller = "TabCompletion", action = "GetMatchingPackages", maxResults = 30, includePrerelease = false });
 
    routes.MapHttpRoute(RouteNames.TabCompletionPackageVersions,
                        "api/v2/package-versions/{packageId}",
                        new {controller = "TabCompletion", action = "GetPackageVersions", includePrerelease = false});
}
 
public static void MapDataServiceRoutes(RouteCollection routes)
{
    var dataServiceHostFactory = new DataServiceHostFactory();
    
    // Maps OData to root of application (NOT /api/v2)
    var serviceRoute = new ServiceRoute("", dataServiceHostFactory, typeof(PackageDataService))
        {
            Defaults = RouteNames.PackageFeedRouteValues,
            Constraints = RouteNames.PackageFeedRouteValues
        };
    
    routes.Add(RouteNames.PackageFeed, serviceRoute);
}
```

Next
----

I would love to see NuGet adopt more RESTful practices with regards to
its HTTP API, but having had one pull request declined already, I'm not sure
how eager I'll be in trying to bring about this change.
