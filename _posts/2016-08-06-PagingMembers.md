---
layout: post
title: "Paging Members from Active Directory"
date:   2016-08-06 20:00:00
comments: true
categories: Active Directory
---

First I'd like to get out of the way that this is my first real blog post.  I am not a regular blogger (clearly) and I don't know how often I'll truly be able to do this, but I desperately want to find ways to share my work and things that I run into on a daily basis so that perhaps others won't have to struggle like I may have.

So that being said I think I've finally found something that I don't see all that well documented that might be worth talking about.  *Most* people would use the Active Directory cmdlets to get group memberships (specifically `Get-ADGroupMember` or `Get-ADGroup -Properties member -Identity groupName`), but there are a couple of scenarios where this might not be ideal so I wanted to try to document how I page out group memberships using `System.DirectoryServices` objects and methods.  I've found a number of scenarios where Microsoft's Active Directory cmdlets don't fit my personal needs and so I've had to write my own AD cmdlets for a number of situations and this is one of them.

The first thing you'll need to do is create a `System.DirectoryServices.DirectorySearcher` object.  If you're familiar with reading MSDN you can find out more about it [here](https://msdn.microsoft.com/en-us/library/system.directoryservices.directorysearcher(v=vs.110).aspx).  Through MSDN's documentation you'll see there are a number of different constructors for a DirectorySearcher object (different ways you can create such an object with certain parameters).  To keep things simple for this post I'm going with an empty constructor.

```powershell
$directorySearcher = New-Object System.DirectoryServices.DirectorySearcher
```
With the searcher created you need to fill out some of the properties before executing out search.  First we'll point it at a specific domain controller, define a proper LDAP filter to find the group, and tell Active Directory what properties we want back:

```powershell
$directorySearcher.SearchRoot = [ADSI]"LDAP://server01.domain.local"
$directorySearcher.Filer = '(&(objectClass=group)(cn=aBigGroup))'
[void]$directorySearcher.PropertiesToLoad.Add('cn')
[void]$directorySearcher.PropertiesToLoad.Add('distinguishedname')
[void]$directorySearcher.PropertiesToLoad.Add('member')
```

Truth be told there is a quicker way to do this with [[ADSISearcher] type accelerator](https://blogs.technet.microsoft.com/heyscriptingguy/2010/08/24/use-the-powershell-adsisearcher-type-accelerator-to-search-active-directory/), however, at least in scripts, I prefer to spell things out in a way that is easy to read.  That being said now that we have our searcher object we can execute our search and store the results in a variable.

```powershell
$results = $directorySearcher.FindAll()
```

Now and interesting thing has happened which you may (or may not) be aware of.  Normally when you make a request to Active Directory for certain properties, if they have values it will return those properties\attributes with the values included:

```powershell
PS c:\> $results.properties.cn -as [String]
aBigGroup
```

The interesting thing about this is that if you look at the member property of this group it is empty!

```powershell
PS c:\> $results.properties.member.count
0
```

Now I happen to know that this group has thousands of members.  The first clue here is to look to see what Active Directory actually returned to us.  If you look up above we specifically asked for cn, distinguishedname, and member.  However if you look at what comes back we got a little surprise:

```powershell
PS c:\> $results.properties.PropertyNames
distinguishedname
member;range=0-1499
member
adspath
cn
```

So we did get back the properties we asked for but we got 2 additional ones.  The adspath is always going to come back with any object you get from Active Directory and is essentially the full path to the object (so this is expected).  The `member;range=0-1499` however might be a little unexpected.  This is the clue to an aware ADSI client that it's time to page out the membership.  If you look at the member;range=0-1499 property you'll see that it has exactly 1,500 distinguished names in it:

```powershell
PS c:\> $results.properties.'member;range=0-1499'.count
1500
```

What the above tells us is that:

1. This is a large group since the range property has been returned.
2. Active Directory is returning a maximum of 1,500 members at a time.

To get at the other members you will need to ask Active Directory for the next batch (1,500 at a time...thus the term "paging").  To see the next batch you need to construct a few things (we'll do it the long way first).

```powershell
$members = New-Object System.Collections.ArrayList
$results.properties.'member;range=0-1499'.Foreach({ [void]$members.Add($psitem) })
$increment = $results.properties.'member;range=0-1499'.count -as [int]

$start = $increment
$end = $start + $increment -1
$memberProperty = 'member;range={0}-{1}' -f $start,$end
$directoryEntry = $results.Properties.adspath -as [String]

$memberPager = New-Object -TypeName System.DirectoryServices.DirectorySearcher -ArgumentList $directoryEntry,'(objectClass=*)',$memberProperty,'Base'
$pageResults = $memberPager.FindOne()
```

There are a couple things going on here worth explaining.  First I created an array list to store the members.  I use an array list (as opposed to an array) because they are a lot faster to add things to so this is more of a efficiency thing.  Next I load into the array the first batch of members that we already got back from AD and then I define an increment based on that initial count of members we got.  By default you will always see 1,500 members returned from AD unless you have custom LDAP policies, but I store that here as a number just to support different page sizes.  Next I define the start and end range.  If our first batch started with 0 and ended with 1499 then the next batch should start with 1500 and end with 1500 + 1500 -1.  That gives us what is stored in `$memberProperty`= member:range=1500-2999.  Finally I pull out the full ADS path of the group and store it as a string since we'll need it as a parameter for our next part.

Next we create another directory searcher object, but this time we structure it a little bit differently.  If you refer back to the MSDN documentation for System.DirectoryServices.DirectorySearcher you'll see that there is a 4 parameter constructor and that's what we're using here.  We're providing it the following:

1. A directory entry in the form of LDAP://<DCName>/<DN of group>.  This is the adspath property that came back from our initial query which we're re-using for the paging.
2. An LDAP filter.  In our case since our directory entry is to a specific object we keep the filter simple.
3. The member property we want to Active Directory to return to us.
4. Finally how far in the directory we should search.  In this case since our directory entry is to a specific object we don't need to search any further so we define it as just base.

With our pager object created we then need to initiate our query for members 1500-2999 by calling the `.FindOne()` method.  I haven't explained it yet, but at this point we know there are more so we just ask for one more.  Now we need to see what AD sent back to us, just like we had done earlier.

```powershell
PS C:\> $pageResults.Properties.PropertyNames
member;range=1500-2999
adspath
```

In this case it gave back exactly what we were expecting so we need to add those members to our member array list:

```powershell
$pageResults.Properties.'member;range=1500-2999'.Foreach({ [void]$members.Add($psitem) })
```

Now when I look at the number of members in the `$members` array list there are 3000.  But we're not done as there are more members to pull out of Active Directory for this group.  Like before we need to augment our start and end range by 1,500 members.  We stopped at 2999 so we need to start with 3000 and we need to end on 4,499 (3000 + 1500 - 1):

```powershell
$start = $end + 1
$end = $start + $increment - 1
$memberProperty = 'member;range={0}-{1}' -f $start,$end
```

We create another DirectorySearcher object like we did before and pull down the next batch of memberships and add them to our array list.  Now you might be asking "how do we know when to stop?".  Interestingly enough Active Directory will tell you.  In the example we're working with the group has a little over 5,000 memberships.  When we ask Active Directory for members ranging from 4500-5999 when the results come back from our query the properties we get will by different than before:

```powershell
PS C:\> $pageResults.Properties.PropertyNames
member;range=4500-*
adspath
```

So we asked for `member;range=4500-5999` but we got back `member;range=4500-*`.  This is Active Directories way of telling us that this is the last batch.  So now we do our last addition to the members array list:

```powershell
$pageResults.Properties.'member;range=4500-*'.Foreach({ [void]$members.Add($psitem) })
```

So as you can see cmdlets such as Get-ADGroupMember or Get-ADGroup are doing quite a bit of heavy lifting for you, however if you are in a situation where you either can't or don't want to use the AD cmdlets to get memberships paging out the members isn't really that hard.

Now what we did above is the ...ahem... manual way and we are talking PowerShell here.  We need to put this thing in a loop to do it right.

```powershell
$directorySearcher = New-Object System.DirectoryServices.DirectorySearcher
$directorySearcher.SearchRoot = [ADSI]'LDAP://DC01.domain.local'
$directorySearcher.Filter = '(&(objectClass=group)(cn=ABigGroup))'
[void]$directorySearcher.PropertiesToLoad.Add('cn')
[void]$directorySearcher.PropertiesToLoad.Add('distinguishedname')
[void]$directorySearcher.PropertiesToLoad.Add('member')
$results = $directorySearcher.FindOne()

$members = New-Object System.Collections.ArrayList
if ($pageProperty = $results.Properties.PropertyNames.Where({$psitem -match '^member:range'}) -as [String]) {
    $directoryEntry = $results.Properties.adspath -as [String]
    $increment = $results.Properties.$pageProperty.count -as [Int]
    $results.Properties.$pageProperty.Foreach({ [void]$members.Add($psitem) })
    $start = $increment
    do {
        $end = $start + $increment - 1
        $memberProperty = 'member;range={0}-{1}' -f $start,$end
        $memberPager = New-Object -TypeName System.DirectoryServices.DirectorySearcher -ArgumentList $directoryEntry,'(objectClass=*)',$memberProperty,'Base'
        $pageResults = $memberPager.FindOne()
        $pageProperty = $pageResults.Properties.PropertyNames.Where({$psitem -match '^member:range'}) -as [String]
        $pageResults.Properties.$pageProperty.Foreach({ [void]$members.Add($psitem) })
    } until ( $pageProperty -match '^member.*\*$' )
}
else {
    $results.Properties.member.Foreach({ [void]$members.Add($psitem) })
}
```

Now all you'd need to do is wrap this up in a function\script, add in some parameters, and a dash of error checking with smart uses of try-catch and you have a group membership tool that doesn't rely on any modules.  Yay for one less dependency!