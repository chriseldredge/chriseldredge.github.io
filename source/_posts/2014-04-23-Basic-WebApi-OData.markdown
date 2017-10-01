---
layout: post
title: "Basic WebApi OData"
date: 2014-04-23 11:28
comments: true
categories: nuget odata
---

Getting started on making a NuGet compatible OData feed with WebApi OData,
we'll start with the simplest parts to get on our feet.

## OData Controller

```c#
public class PackagesODataController : ODataController
{
    public IMirroringPackageRepository Repository { get; set; }

    [Queryable(PageSize = 100, HandleNullPropagation = HandleNullPropagationOption.False)]
    public IQueryable<ODataPackage> Get()
    {
        return Repository.GetPackages().Select(p => p.ToODataPackage()).AsQueryable();
    }
}
```

This controller exposes an entity set of package metadata. The underlying
entity type `IPackage` has some complex types that OData doesn't play well
with, so the types are decorated with `ToODataPackage` which flattens
and simplifies the metadata into primitive types like strings, bools
and ints.

The `QueryableAttribute` exposes some very useful settings that allow us to
enable or disable advanced querying options like `$select`, `$expand`,
`$orderby` and others that can be used to lock down the endpoint to avoid
overly expensive or invalid queries from being executed.

`HandleNullPropagation` is a much welcomed offering that wasn't easily
tweakable before. It tells the engine whether or not the underlying query
execution engine can handle null values in expressions. The default is
to modify query expressions dynamically to put null-check conditions around
all comparisons and operations like `ToLowerCase()`, `Contains()` and such.
This is necessary for LINQ to Collections, but generally not necessary for
other LINQ implementations.

Null Propagation seems like a prudent default,
but it can cause performance problems. David Ebbo [wrote](http://blog.davidebbo.com/2011/08/how-odata-quirk-killed-nuget-server.html)
about an issue the NuGet Gallery team encountered. If your LINQ provider
doesn't need it, you should definitely turn it off.

## Configuration

```c#
public void MapDataServiceRoutes(HttpConfiguration config)
{
    var builder = new ODataConventionModelBuilder();

    var entity = builder.EntitySet<ODataPackage>("Packages");
    entity.EntityType.HasKey(pkg => pkg.Id);
    entity.EntityType.HasKey(pkg => pkg.Version);

    var conventions = ODataRoutingConventions.CreateDefault()
        .Select(c => (IODataRoutingConvention)
          new ControllerAliasingODataRoutingConvention(
            c, "Packages", "PackagesOData"));

    config.Routes.MapODataRoute(
        RouteNames.Packages.Feed,
        ODataRoutePath,
        builder.GetEdmModel(),
        new DefaultODataPathHandler(),
        conventions);
}
```

This is mostly a vanilla configuration but already there are two modifications.

First, the `Packages` entity set has a composite key that uniquely identifies
each entity instance, comprised of a package ID and version.

The next tweak is that we're customizing the default list of OData routing
conventions, wrapping each one with `ControllerAliasingODataRoutingConvention`.

My project already has a `PackagesController` that inherits from `ApiController`.
I wanted to put OData related methods in a separate controller and name it
`PackagesODataController`, but still have my entity set be named `Packages`.

```c#
public class ControllerAliasingODataRoutingConvention : IODataRoutingConvention
{
    private readonly IODataRoutingConvention delegateRoutingConvention;
    private readonly string controllerAlias;
    private readonly string targetControllerName;

    public ControllerAliasingODataRoutingConvention(
        IODataRoutingConvention delegateRoutingConvention,
        string controllerAlias,
        string targetControllerName)
    {
        this.delegateRoutingConvention = delegateRoutingConvention;
        this.controllerAlias = controllerAlias;
        this.targetControllerName = targetControllerName;
    }

    public string SelectController(ODataPath odataPath, HttpRequestMessage request)
    {
        var controller = delegateRoutingConvention.SelectController(odataPath, request);
        return string.Equals(controller, controllerAlias,
                StringComparison.OrdinalIgnoreCase)
            ? targetControllerName
            : controller;
    }

    public string SelectAction(
        ODataPath odataPath,
        HttpControllerContext controllerContext,
        ILookup<string, HttpActionDescriptor> actionMap)
    {
        return delegateRoutingConvention.SelectAction(
          odataPath, controllerContext, actionMap);
    }
}
```

This is a basic decorator that renames a standard controller name with a desired
target name.

## Summary

At this point we have a WebApi endpoint that speaks OData. We can enumerate
packages, filter, order and page. We're still pretty far from something that
a NuGet client can speak to though.

{% include custom/nuget_webapi_odata_series_index.html %}
