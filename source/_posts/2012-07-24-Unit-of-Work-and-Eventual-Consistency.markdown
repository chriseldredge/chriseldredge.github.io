---
layout: post
title: Unit of Work & Eventual Consistency
---

_Lucene.Net.Linq is available on the NuGet Gallery at [http://nuget.org/packages/Lucene.Net.Linq](http://nuget.org/packages/Lucene.Net.Linq)._

Recent versions of Lucene.Net.Linq added support for the [Unit of Work](http://martinfowler.com/eaaCatalog/unitOfWork.html) pattern.

The standard Unit of Work has methods like <code>registerNew</code> and <code>registerDeleted</code>, but I decided to use more generic
names that make the interface appear more like a simple collection of documents.

{% codeblock lang:csharp %}
public interface ISession<T> : IDisposable
{
	IQueryable<T> Query();
	void Add(params T[] items);
	void Delete(params T[] items);
	void Delete(params Query[] items);
	void DeleteAll();
	void Commit();
	void Rollback();
}
{% endcodeblock %}

The <code>Delete</code> method that takes one or more <code>Query</code> objects is a case special to Lucene.Net. This is an escape hatch
for when you want to delete one or more documents without first retrieving them. For example, you may want to delete all documents that
match a query like <code>type:person</code>. I have mixed feelings about this escape hatch, because it smells like a
[Leaky Abstraction](http://c2.com/cgi/wiki?LeakyAbstraction). On the other hand, since it is an overload to a more abstract method,
I think it makes sense from a performance standpoint.

Anyway, one notable method pair is missing from the interface: <code>registerDirty</code> and <code>registerClean</code>. That's because
the session does some book keeping behind the scenes to automatically detect dirty documents when the session is committed.

This makes using Lucene.Net.Linq as a [Repository](http://martinfowler.com/eaaCatalog/repository.html) dead simple, and makes the code
look nice and clean in the client:

{% codeblock lang:csharp %}
public void ChangeMailingAddress(string name, string newAddress)
{
    var session = provider.OpenSession<SampleDocument>();
            
    using (session)
    {
	    var person = session.Query().Single(p => p.Name == name);
	    person.MailingAddress = newAddress;
    }
}
{% endcodeblock %}

## Document Tracking ##

So how does it work?

When you ask for an IQueryable from session, the session attaches an instance of IRetrievedDocumentTracker to the queryable before
returning it. That internal interface has only one method call:

{% codeblock lang:csharp %}
void TrackDocument(T item, T hiddenCopy);
{% endcodeblock %}

When the LuceneQueryExecutor starts returning results one item at a time, if it detects that a document tracker has been attached,
it makes two copies of each result. One is returned to the client and the other is passed only to the tracker as a hidden copy.
This gives the library the ability to compare pristine objects that came from the index unmodified with ones that may have been
modified with the client.

{% codeblock lang:csharp %}
for (var i = skipResults; i < hits.ScoreDocs.Length; i++)
{
	var doc = hits.ScoreDocs[i].doc;
	var score = hits.ScoreDocs[i].score;
	
	var item = ConvertDocument(searcher.Doc(doc), score);

	if (tracker != null)
	{
		var copy = ConvertDocument(searcher.Doc(doc), score);
		tracker.TrackDocument(item, copy);
	}

	itemHolder.Current = item;
	yield return projector(itemHolder.Current);
}
{% endcodeblock %}

Finally, when the session is committed, all tracked documents are compared with their hidden copies (using reflection) to detect
which documents were modified. Modified documents are written back to the index.

## Eventual Consistency ##

The concepts of eventual consitency describe what expectations a client may have about the ordering and visibility of writes in
a distributed asynchronous persistence system. Werner Vogels has written about different
[consistency definitions](http://www.allthingsdistributed.com/2008/12/eventually_consistent.html).

Since the unit of work in Lucene.Net.Linq is implemented by an interface named ISession, it would have been nice to provide
Session Consistency. It's open to some interpretation if this behavior is achieved or not. Since the underlying Lucene engine
will not make changes visible to an IndexReader until a commit happens and the IndexReader is reopened, it is difficult or
impossible to allow a client to see the effects of adding, deleting or modifying a document in subsequent queries within the same
session. However, this problem can be side-stepped by simply calling <code>Commit</code> between making changes and subsequent queries.
Calling <code>Commit</code> explicitly may have drawbacks if the client will make further changes and wishes for those changes to
appear atomically. Knowing the details of when staged changes become visible will help clients to make changes smartly.


