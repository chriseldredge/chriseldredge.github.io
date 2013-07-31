---
layout: post
title: "Putting the web into WebApi"
date: 2013-07-30 22:05
comments: true
categories: 
---

The .net api developers, the WebApi framework offers a compelling alternative
to asp.net mvc for slinging json or (gasp) xml over http. One great feature
you get out of the box is content negotiation, where WebApi will look at
request headers to decide which format to use when processing a request
body or sending back a response.

WebApi also includes the concept of an ApiExplorer, a built in service that
knows about all the routes, templates and parameters an application is
configured to handle. It even supports attaching human friendly documentation.

With these built in perks, WebApi gets you 95% of the way there. What it
doesn't do out of the box, unfortunately, is something that it says right
on the box. It doesn't do web.

HTTP, ReST and the Web
----------------------

There's been a growing buzz around the idea that writing an application
that speaks json does not give you something that can be called ReST.
To get to ReST nirvana your api needs to speak with hypermedia, giving
the client, in addition to data, links and forms that can do stuff with
the data.

If you aren't familiar with this distinction, here are some resources
that sell the concept well:

* [Richardson Maturity Model](http://martinfowler.com/articles/richardsonMaturityModel.html)
* [Building Hypermedia APIs with HTML](http://www.infoq.com/presentations/web-api-html) (talk by Jon Moore)
* [Many slides by Stefan Tilkov](https://speakerdeck.com/stilkov)
* [Exist In the Web - Not on it](https://speakerdeck.com/ammeep/exist-in-the-web-not-on-it) by Amy Palamountain

Microdata
---------

Jon Moore's work is particularly intriguing because he shows what
you can already do with [Microdata](http://en.wikipedia.org/wiki/Microdata_(HTML)).

Microdata boils down to 3 attributes that you stick on elements in your
html5 markup, and they're all you need to give a computer semantic
data. You can nest properties into a scope, describe the type of data
being represented, and even nest scopes with scopes recursively.

Jon's talks show how a web client that speaks Microdata can already start
on sites like [ebay.com](http://ebay.com/) and find and manipulate resources
without knowing anything ahead of time about what resources are there and
what URLs are used to access them.

WebApi + Microdata
------------------

On the one hand we have a .net framework for writing HTTP apis that
has pluggable support for different mime types. On the other hand we
have html5 and Microdata. Think they could be friends?

I'm excited to announce HtmlMicrodataFormatter,
a new open source (APL) project available
on [nuget.org]()
and hosted on [GitHub]().

When you install this package into your WebApi project, it adds two
features.

First, if you enter the URL for one of your api methods into a web
browser, you get (relatively) nice, human friendly html5 back instead
of the wall of xml you would otherwise be greeted with.

Second, and perhaps more importantly, if you browse to ~/api,
you will be greeted by an html5 page that includes links and
forms for all the routes that are registered with ApiExplorer.

If your routes are not templated, you can fill in the inputs
and submit forms to test them out right in the browser. If your
routes are templated, you can still use the forms in the browser,
but you'll need to do some extra work to fill in the template
and update the form action in javascript before the form is
submitted. A future post will cover how this can be done.

Turn on xml documentation for your project, rebuild, and now
all the documentation comments you attach to your controllers
and actions appear on the forms.

Why it Matters
--------------

Obviously, getting a free browser based form generated for
each action can be a productivity boon for developers who
are tired of writing

    curl -X DELETE -H 'Accept: application/json' http://localhost:49497/api/users/fred'
    
But it's more than that. Make your client project use html5
and instead of hard-coding URLs for each thing the client
wants to do, it enters at /api, finds the form it wants,
fills it in and submits it. The Microdata attributes in
the response, along with links and forms, give the client
the pieces it needs to drive state, just like Roy T. Fielding
intended when ReST was first described.

Have third party clients that consume your api? Now you
don't have to give them a separate document or website
that describes (out of band) how to use your api. Just
link them to yourwebsite/api, and they can read the
documentation and discover what they can do without
guessing or following brittle conventions.

Putting the web into WebApi removes the tight coupling
between client and server. Now you can change your routes,
rename parameters, add parameters, and the client will
automatically see those changes reflected in the forms
and links they see when they enter your api. No more `/api/v1/users`
through `/api/v23/users`. No more api version headers.

Next Steps
----------

HtmlMicrodataFormatter is pretty young and there are
many ideas left to flesh out. For example, making it easy to add
links and forms to the output of controller actions,
or improving how xml documentation is collected and
converted to html, or making it easier to link from
resources to related resources with appropriate rel
attributes, or, or, or…

I hope you like HtmlMicrodataFormatter, and if you
do, maybe get involved and help improve it by
submitting feedback, bugs, enhancements or pull
requests to the GitHub project.

Similar to Jon Moore's python based microdata
client, work has started on a .net dynamic
client which I hope to share soon.

