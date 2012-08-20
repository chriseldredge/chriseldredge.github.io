---
layout: post
title: Lucene NumericField Gotcha
comments: true
categories: 
---

Today I ran into an interesting problem using
[NumericField](https://svn.apache.org/repos/asf/incubator/lucene.net/trunk/src/core/Document/NumericField.cs).

The issue is that when a document is retrieved, fields encoded using NumericField come back as plain old
strings.  This isn't a problem by itself, but if you retrieve a document, modify some fields and leave
others alone, any unmodified field that was previously a NumericField reverts to a basic string value.

The issue is demonstrated by this sample NUnit test fixture:
<script src="https://gist.github.com/2425347.js"> </script>

Long story short, if your documents are going to take round trips and get re-indexed, you must re-encode
any NumericFields before adding the document back to the index, even if they did not change.

