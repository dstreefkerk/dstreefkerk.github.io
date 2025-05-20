---
layout: post
title: "Correct Horse Battery Staple for PowerShell, AKA Random Memorable Password Generator"
date: 2016-05-18
categories: 
  - powershell
  - security
tags:
  - invoke-webrequest
  - passwords
  - powershell
  - systems-administration
author: "Daniel Streefkerk"
excerpt: "Creating a PowerShell function to generate memorable passwords based on the XKCD 'correcthorsebatterystaple' concept - a simple approach to generating random but memorable passwords using common words."
---

> **Note:** This article was originally written in May 2016 and is now over 9 years old. Due to changes in PowerShell and security practices over time, the described solution may no longer work as written. Please consider this guide as a conceptual reference rather than a current implementation guide.

I'm a fan of using [correcthorsebatterystaple.net](http://correcthorsebatterystaple.net/) for generating temporary passwords for users, rather than always using a static password. The site itself is a reference to the XKCD [webcomic](https://xkcd.com/936/), and yes, I'm aware that there are [plenty](https://diogomonica.com/posts/password-security-why-the-horse-battery-staple-is-not-correct/) [of](http://hackaday.com/2015/10/26/a-more-correct-horse-battery-staple/) [opinions](https://www.reddit.com/comments/2j7jvr) on the web about this topic.

*[Image removed during migration: XKCD comic strip showing password strength comparison between a complex but hard to remember password like "Tr0ub4dor&3" and a sequence of four random common words like "correct horse battery staple", demonstrating that the latter has more entropy while being easier to remember]*

I've had the idea in the back of my mind for a while to see if I could replicate the site's functionality in PowerShell. I noticed that the source code for the site is on [GitHub](https://github.com/jvdl/CorrectHorseBatteryStaple), so I ducked over there to check out the [word list](https://github.com/jvdl/CorrectHorseBatteryStaple/blob/master/data/wordlist.txt).

I found that it's possible to replicate most of the functionality of the site with just two lines of PowerShell (although it doesn't result in very readable code):
1. I used [Invoke-WebRequest](https://technet.microsoft.com/en-us/library/hh849901.aspx) to grab the word list from GitHub
2. I then expanded out the *Content* property, and split it up given the comma delimiter
3. I then used a combination of the [Range operator](https://technet.microsoft.com/en-us/library/hh847732.aspx), [Foreach-Object](https://technet.microsoft.com/en-us/library/hh849731.aspx), [[string]::join](https://msdn.microsoft.com/en-us/library/57a79xd0(v=vs.110).aspx), [Get-Random](https://technet.microsoft.com/en-us/library/hh849905.aspx) and the [TextInfo class](https://msdn.microsoft.com/en-au/library/system.globalization.textinfo.totitlecase(v=vs.110).aspx) to generate a given number of passwords along these rules:
   1. 4 random words, each with the first letter capitalised
   2. A separator in between
   3. A random number between 1 and 99 at the end

Note that this isn't failure-proof, and isn't intended to be used in any complex scenario. There's no error handling, and not much flexibility built in. It's just a quick function you could put into your PowerShell profile.

You can, at least, do the following:
*[Image removed during migration: Screenshot of PowerShell ISE window showing the output of Get-RandomPassword command. The screenshot displayed several generated passwords using the function, with examples of using different separators and count parameters]*

Here's the code:

<script src="https://gist.github.com/dstreefkerk/0e2e78bb16edcc78467e0dbc75ad0a9a.js"></script>