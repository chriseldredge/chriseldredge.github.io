---
layout: post
title: "WinDbg: Dump entries in array"
date: 2014-03-05 11:47
comments: true
categories: windbg
---

I was debugging a stack overflow exception in some c# code
today and used this recipe to dump the request headers
associated with the offending thread.

I'm leaving it here for my own reference.

This recipe dumps the key and value fields in an array of
NameObjectCollectionBase+NameObjectEntry, though it would
probably work with similar key/value tuples.


```
.foreach /pS 16 /ps 1 (addr {!da 3ffc0c460}) {
  .if ($sicmp("${addr}", "null") != 0) {
    !do poi(${addr}+8); !do poi(${addr}+10)
  }
}
```

_Note: the sample code above is formatted for readability. Remove
line breaks before using it in windbg, which doesn't seem to handle
line breaks in commands well._

The DumpArray command in SOS does not have a `-short` flag like
DumpHeap does, so the foreach skips the first 16 tokens, then
skips a single token for each item processed. This effectively
selects array item addresses from the output.

Inside the foreach block, I check that the address is not `null`,
then dump the fields at offsets 8 and 16 (0x10 in hex).
