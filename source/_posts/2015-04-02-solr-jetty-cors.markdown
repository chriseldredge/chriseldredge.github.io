---
layout: post
title: "Solr, Jetty and CORS"
date: 2015-04-02 10:29
comments: true
categories: 
---

Recently my pet project has been working on an [Ember Data](https://github.com/emberjs/data) adapter that connects to Solr. This work combines two of my favorite technologies and I think has a lot of potential value for creating applications
quickly and easily in Ember that leverage the full power of Solr as a persistence
and retrieval engine. My fledgling project lives at [github.com/chriseldredge/ember-solr](https://github.com/chriseldredge/ember-solr). It's heavily based on EmberFire.

One issue I ran into, of course, is dealing with cross-origin requests in the browser.
For my tests, my Ember app is running on localhost:4200 and my Solr server is running
on localhost:8983. The browser, rightly, will refuse to let requests from my Ember app
execute on Solr, unless the Solr server enables [Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS) headers.

Solr 5 ships with Jetty, and it turns out there's already a servlet filter that
can do this for us. All we have to do is download it and configure it.

First, figure out which version of Jetty your Solr shipped with. Solr 5.0 comes with
Jetty 8.1.10. Now head over to the [jetty-servlets](http://repo1.maven.org/maven2/org/eclipse/jetty/jetty-servlets/) folder on the maven.org repository, drill down into
the corresponding version, and download jetty-servlets-_version_.jar. Save the jar
into `$SOLR_HOME/server/lib`.

Next, open `$SOLR_HOME/server/etc/webdefault.xml` in a text editor.
Insert the following at bottom of the file, right before the closing `</web-app>`
tag:

```xml
  <filter>
    <filter-name>cross-origin</filter-name>
    <filter-class>org.eclipse.jetty.servlets.CrossOriginFilter</filter-class>
    <init-param>
      <param-name>chainPreflight</param-name>
      <param-value>false</param-value>
    </init-param>
  </filter>

  <filter-mapping>
    <filter-name>cross-origin</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

The third and final step is to start (or restart) your Solr server.

This example enables CORS requests for all origins. If you are going to do something
like this on your production server, you probably want to lock this down using the
`allowedOrigins` parameter.

The `chainPreflight` parameter is set to false so the
filter will short-circuit when a browser sends a pre-flight OPTIONS request.
This is necessary for Solr because the Solr update handler would respond with
a `400 Bad Request` since it doesn't know what to do with an OPTIONS request.

You can find the other configuration parameters for the filter at [wiki.eclipse.org](https://wiki.eclipse.org/Jetty/Feature/Cross_Origin_Filter).

## Alternatives

If you can't or don't want to use CORS, there are a couple other options available
to you. You could have your Ember app proxy requests to Solr, you could deploy your
Ember app on the same origin as your Solr server, or you could try to use JSON-P.

