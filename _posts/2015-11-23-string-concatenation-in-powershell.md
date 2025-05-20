---
layout: post
title: "String Concatenation in PowerShell"
date: 2015-11-23
categories: [powershell]
tags: [powershell, string-formatting]
author: "Daniel Streefkerk"
excerpt: "Learn more efficient ways to concatenate strings in PowerShell using expanding strings and the format operator instead of the traditional concatenation method."
---

> **Note:** This article was originally written in November 2015 and is now over 9 years old. Due to changes in PowerShell over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

I was watching a Pluralsight course this morning, and I came across the second instance of VBA-style string concatenation that I'd seen this week.

You know, like this:

```powershell
$FirstName = "Joe"
$LastName = "Bloggs"
$DisplayName = $LastName + ", " + $FirstName
$UserName = $FirstName + "." + $LastName + "@globomantics.org"
```

This is one of my pet peeves, as it's not necessary in PowerShell, and it makes the code harder to read.

PowerShell is quite capable of expanding variables within a string. This is called an 'expanding string':

```powershell
$DisplayName = "$LastName, $FirstName"
$UserName = "$FirstName.$LastName@globomantics.org"
```

If you want to go one step further, you can use the format operator:

```powershell
$DisplayName = "{0}, {1}" -f $LastName,$FirstName
$UserName = "{0}.{1}@globomantics.org" -f $FirstName,$LastName
```

This allows for cleaner insertion of variables, especially if you want to do something like this to get rid of capital letters in an email address or UPN (another of my pet peeves):

```powershell
$UserName = "{0}.{1}@globomantics.org" -f $FirstName.ToLower(),$LastName.ToLower()
```

Some good reading on the Microsoft PowerShell Scripting Blog (formerly "Hey, Scripting Guy!"):

* [Understanding PowerShell and Basic String Formatting](https://devblogs.microsoft.com/scripting/understanding-powershell-and-basic-string-formatting/)
* [Use PowerShell to Format Strings with Composite Formatting](https://devblogs.microsoft.com/scripting/use-powershell-to-format-strings-with-composite-formatting/)

## Additional Notes, 2025

### Modern PowerShell Alternatives

PowerShell 5+ also allows for interpolation using sub-expressions for more complex expressions. I still prefer the format operator though:

```powershell
$UserName = "$($FirstName.ToLower()).$($LastName.ToLower())@globomantics.org"
```

### ### Performance Considerations

For high-volume string operations:

* The format operator (`-f`) can be more efficient than string concatenation with `+`
* Consider using `StringBuilder` from the .NET Framework for intensive string manipulations

Remember to follow consistent formatting practices in your PowerShell scripts to improve readability and maintainability.