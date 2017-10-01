---
layout: post
title: "Default Streams with WebApi OData"
date: 2014-04-29 16:07
comments: true
categories: nuget odata
---

NuGet package feeds provide metadata about packages, but after searching
for new packages to use or updates to existing ones, we also have to
be able to download the actual package.

The NuGet OData implementation provides links to download a package
by using the `default stream` concept.

In WCF Data Services, an entity indicates that it has a stream using the
aptly named `HasStreamAttribute`.

```c#
[DataServiceKey("Id", "Version")]
[HasStream]
public class DataServicePackage : IEquatable<DataServicePackage>
{
    //snip
}
```

To build a URI for each instance, a custom implementation of
`IDataServiceStreamProvider` is wired into our `DataService`.

This is done differently in WebApi which does not include the
`HasStreamAttribute` at all.

Over a year ago, I asked on twitter:

{% tweet https://twitter.com/creldredge/status/289384233141669888 %}

At the time, the answer was no. However, I'm happy to report that
recent versions of WebApi do support this feature.

First, we need to tell our EDM model builder that the entity has
a default stream:

```c#
public void MapODataRoutes(HttpConfiguration config)
{
    var builder = new ODataConventionModelBuilder();

    var entity = builder.EntitySet<ODataPackage>("Packages");
    entity.EntityType.HasKey(pkg => pkg.Id);
    entity.EntityType.HasKey(pkg => pkg.Version);

    var entityType = model
      .FindDeclaredType(typeof(ODataPackage).FullName);

    model = builder.GetEdmModel();
    model.SetHasDefaultStream(
       entityType as IEdmEntityType, hasStream: true);

    // snip
}
```

Next we'll need to implement a custom `ODataSerializerProvider`
and `ODataEdmTypeSerializer` to provide a default stream URL for
entity instances:

```c#
public class ODataPackageDefaultStreamAwareSerializerProvider
    : DefaultODataSerializerProvider
{
    private readonly ODataEdmTypeSerializer entitySerializer;

    public ODataPackageDefaultStreamAwareSerializerProvider()
    {
        this.entitySerializer =
          new ODataPackageDefaultStreamAwareEntityTypeSerializer(this);
    }

    public override ODataEdmTypeSerializer GetEdmTypeSerializer(
      IEdmTypeReference edmType)
    {
        if (edmType.IsEntity())
        {
            return entitySerializer;
        }

        return base.GetEdmTypeSerializer(edmType);
    }
}

public abstract class DefaultStreamAwareEntityTypeSerializer<T>
    : ODataEntityTypeSerializer where T : class
{
    protected DefaultStreamAwareEntityTypeSerializer(
      ODataSerializerProvider serializerProvider)
        : base(serializerProvider)
    {
    }

    public override ODataEntry CreateEntry(
      SelectExpandNode selectExpandNode,
      EntityInstanceContext entityInstanceContext)
    {
        var entry = base.CreateEntry(selectExpandNode,
          entityInstanceContext);

        var instance = entityInstanceContext.EntityInstance as T;

        if (instance != null)
        {
            var link = BuildLinkForStreamProperty(
              instance, entityInstanceContext);

            entry.MediaResource = new ODataStreamReferenceValue
            {
                ContentType = ContentType,
                ReadLink = new Uri(link)
            };
        }
        return entry;
    }

    protected virtual string ContentType
    {
        get { return "application/octet-stream"; }
    }

    protected abstract string BuildLinkForStreamProperty(
      T entity,
      EntityInstanceContext entityInstanceContext);
}

public class ODataPackageDefaultStreamAwareEntityTypeSerializer : DefaultStreamAwareEntityTypeSerializer<ODataPackage>
{
    public ODataPackageDefaultStreamAwareEntityTypeSerializer(
      ODataSerializerProvider serializerProvider)
      : base(serializerProvider)
    {
    }

    protected override string BuildLinkForStreamProperty(
      ODataPackage package,
      EntityInstanceContext context)
    {
        var url = new UrlHelper(context.Request);
        var routeParams = new { package.Id, package.Version };
        return url.Link(RouteNames.Packages.Download, routeParams);
    }

    protected override string ContentType
    {
        get { return "application/zip"; }
    }
}
```

The most relevant part above is the override of `CreateEntry`
(line 33) which sets the `MediaResource` property on the
serialized `ODataEntry`.

Finally the custom serializer is wired into the configuration:

```c#
config.Formatters.InsertRange(0,
    ODataMediaTypeFormatters.Create(
        new ODataPackageDefaultStreamAwareSerializerProvider(),
        new DefaultODataDeserializerProvider()));
```

This whole mess of code nets us a modest customization in the Atom
output from WebApi OData:

```xml
<entry>
  <id>http://example/api/odata/Packages(Id='Lucene.Net.Linq',Version='3.2.55')</id>
  <content
    type="application/zip"
    src="http://example/api/packages/Lucene.Net.Linq/3.2.55/content"/>
  <!-- snip -->
</entry>
```

This enables the NuGet client to download packages.

{% include custom/nuget_webapi_odata_series_index.html %}
