---
layout: post
title: "Composite Keys with WebApi OData"
date: 2014-04-24 11:28
comments: true
categories: nuget odata
---

In our basic configuration we told the model builder that
our entity has a composite key comprised of an ID and a
version:

```c#
public void MapDataServiceRoutes(HttpConfiguration config)
{
    var builder = new ODataConventionModelBuilder();

    var entity = builder.EntitySet<ODataPackage>("Packages");
    entity.EntityType.HasKey(pkg => pkg.Id);
    entity.EntityType.HasKey(pkg => pkg.Version);

    // snip
}
```

This is enough for our OData feed to render `edit` and `self`
links for each individual entity in a form like:

    http://localhost/odata/Packages(Id='Sample',Version='1.0.0')

But if we navigate to this URL, instead of getting just this one
entity by key, we get back the entire entity set.

To get the correct behavior, first we need an override on our
PackagesODataController that gets an individual entity instance
by key:

```c#
public class PackagesODataController : ODataController
{
    public IMirroringPackageRepository Repository { get; set; }

    public IQueryable<ODataPackage> Get()
    {
        return Repository.GetPackages().Select(p => p.ToODataPackage()).AsQueryable();
    }

    public IHttpActionResult Get(
        [FromODataUri] string id,
        [FromODataUri] string version)
    {
        var package = Repository.FindPackage(id, version);

        return package == null
          ? (IHttpActionResult)NotFound()
          : Ok(package.ToODataPackage());
    }
}
```

However, out of the box WebApi OData doesn't know how to bind
composite key parameters to an action such as this, since
the key is comprised of multiple values.

We can fix this by creating a new routing convention that
binds the stuff inside the parenthesis to our route data map:

```c#
public class CompositeKeyRoutingConvention : IODataRoutingConvention
{
    private readonly EntityRoutingConvention entityRoutingConvention =
        new EntityRoutingConvention();

    public virtual string SelectController(
        ODataPath odataPath,
        HttpRequestMessage request)
    {
        return entityRoutingConvention
          .SelectController(odataPath, request);
    }

    public virtual string SelectAction(
        ODataPath odataPath,
        HttpControllerContext controllerContext,
        ILookup<string, HttpActionDescriptor> actionMap)
    {
        var action = entityRoutingConvention
            .SelectAction(odataPath, controllerContext, actionMap);

        if (action == null)
        {
            return null;
        }

        var routeValues = controllerContext.RouteData.Values;

        object value;
        if (!routeValues.TryGetValue(ODataRouteConstants.Key,
          out value))
            {
              return action;
            }

        var compoundKeyPairs = ((string)value).Split(',');

        if (!compoundKeyPairs.Any())
        {
            return null;
        }

        var keyValues = compoundKeyPairs
            .Select(kv => kv.Split('='))
            .Select(kv =>
              new KeyValuePair<string, object>(kv[0], kv[1]));

        routeValues.AddRange(keyValues);

        return action;
    }
}
```

This class decorates a standard `EntityRoutingConvention`
and splits the raw key portion of the URI into key/value pairs
and adds them all to the routeValues dictionary.

Once this is done the standard action resolution kicks in
and finds the correct action overload to invoke.

This routing convention was adapted from the WebApi
[ODataCompositeKeySample](http://aspnet.codeplex.com/SourceControl/changeset/view/9cb7243bd9fe3b2df484bf2409af943f39533588#Samples/WebApi/ODataCompositeKeySample/ODataCompositeKeySample/Extensions/CompositeKeyRoutingConvention.cs)
project.

Here we see another difference between WebApi OData and WCF
Data Services. In WCF Data Services, the framework handles
generating a query that selects a single instance from
an `IQueryable`. This limits our ability to customize
how finding an instance by key is done. In WebApi OData,
we have to explicitly define an overload that gets an
entity instance by key, giving us more control over
how the query is executed.

This distinction might not matter for most projects, but
in the case of [NuGet.Lucene.Web](https://github.com/themotleyfool/NuGet.Lucene/tree/master/source/NuGet.Lucene.Web),
it enables a mirror-on-demand
capability where a local feed can fetch a package from another
server on the fly, add it to the local repository, then
send it back to the client as if it was always there in the
first place.

To customize this in WCF Data Services required
significant [back](https://github.com/themotleyfool/NuGet.Lucene/blob/v2.9.4/source/NuGet.Lucene.Web/DataServices/PackageDataSource.cs#L17)
[flips](https://github.com/themotleyfool/NuGet.Lucene/blob/v2.9.4/source/NuGet.Lucene.Web/DataServices/PackageDataService.cs#L171).

{% include custom/nuget_webapi_odata_series_index.html %}
