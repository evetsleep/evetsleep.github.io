---
layout: post
title: "Paging Members from Active Directory"
date:   2016-08-06 20:00:00
comments: true
categories: adsi
---

I use [System.DirectoryServices](https://msdn.microsoft.com/en-us/library/system.directoryservices(v=vs.110).aspx) all the time in my PowerShell scripts (I have a lot of client machines which don't have RSAT installed so I use System.DirectoryServices.DirectorySearcher instead) and for some reason it never occurred to me to ask this question. When I query a directory (Active Directory, OpenLDAP, or something else) how exactly does .NET know that the cn attribute is a string or that proxyAddresses is an array of strings? Since it just works I never thought to look into it, however as is almost always the case I was forced to ask and answer the question when a number of my scripts (and some .NET applications) stopped working properly when querying a non-AD directory.

The problem was simply this. When querying using S.DS.DS (System.DirectoryServices.DirectorySearcher) almost (but not all) results were coming back as an array of bytes instead of their proper types (strings, integers, booleans, etc..). Some attributes were however working.

<img src="/static/img/_posts/sd_byte_array01.png" class="center" />

Without going into too much detail sd is a PowerShell function that performs S.DS.DS searches against a variety of directories (AD and otherwise). Normally the output would be strings (or an array of strings), but instead as you can see its returning byte arrays. I had to remove the private stuff, but you can see from this that some attributes were properly being rendered, but most were not. This is not normal.

I fired up [WireShark](https://www.wireshark.org/) to take a look at was going on and the path to S.DS.DS enlightenment began. What I noticed was that while capturing LDAP queries with .Net it was doing something that was unexpected. It would go something like this:

1. Bind to the directory server (bindRequest, bindResult, searchRequest for RootDSE, searchResEntry, and finally searchResDone).
2. Perform the actual search (searchRequest, searchResEntry, and then searchResDone).
3. But this is what surprised me, under the hood it then does another query looking for `(objectClass=subSchema)` and asking for the modifyTimeStamp.


What was fascinating about #2 is that in the search results the requested attributes were **NOT** coming down as byte arrays, but as their proper string types. So it was S.DS.DS that was not rendering it right. And #3 was kind of interesting because I wasn't expecting it to do any sort of schema queries. I didn't know that this was a thing, but as it turns out some LDAP clients do this.

We'll get back to #2 in a moment, but what was happening when it was querying for the modifyTimeStamp of the subSchema was part of S.DS.DS's schema caching mechanism. What is going on here is that when S.DS.DS executes this query it compares the modifyTimeStamp with the Time value in the registry (if it exists) at `HKCU\SOFTWARE\Microsoft\ADs\Providers\LDAP\schema` DN. For example:

<img src="/static/img/_posts/registry_example.png" class="center" />

In this case it's the Active directory schema and you can see that it gives a file path and the Time. The value in Time is **exactly** the time that comes down from the directory server when the subSchema was queries. If the lastModifyTime from the schema is more recent (or there isn't an entry in the registry) S.DS.DS creates a new cache file and updates the registry. With WireShark I saw this in action where after it queried the directory servers subSchema it then executed another query asking for all objects in the schema and return their attributeTypes, objectClasses, and ditContentRules (this is AD specific, but it asks for it anyways).

If you look at the schema cache file it's pretty interesting stuff (for me anyways):

<img src="/static/img/_posts/schema_cache.png" class="center" />

This actually answered a question that I never thought to ask. S.DS.DS queries the schema and if a newer one is found it creates a cache file which it loads into memory so that it knows how to render data that comes down from the directory server (it does this from any kind of LDAP server).

Now that we understand how this works we can get back to #2...our problem. What was interesting is that the registry key\cache wasn't being generated for the problematic directory server. I could point S.DS.DS at any other directory infrastructure we have (AD or otherwise) and it was using\generating cache files, but this one system was problematic. Going back to WireShark and comparing known good behavior with bad it was clear that in the problem interaction S.DS.DS was issuing a query for the subSchema and actually getting the results back, however it never got a proper LDAP response to the query. I would see the searchRequest and all the responses at the TCP layer (reassembled):

<img src="/static/img/_posts/packet_cap.png" class="center" />

But I would never get a completed searchResEntry or searchResDone. Looking through the actual package bytes the very last one always stopped on the same schema attribute, so I asked the guys that managed this particular directory service about it and they had recently had a problem with that schema attribute and for some reason we were not getting a complete result at the network level from the directory server due to this schema *corruption*. The reason that S.DS.DS was able to render some attirbutes is that it knows that some schema objects are standardized and it knows what format they should be in. For example, it knows that cn, givenname, initials, and l are all strings which explains why they were rendered properly while everything else looked like a byte array. S.DS.DS didn't know how to render all of these other attributes because it has no cache file as a reference! The directory service admins restored the schema and S.DS.DS could now properly render the attributes to their proper type AND we all learned a little more about how System.DirectoryServices.DirectorySearcher works.