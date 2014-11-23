---
layout: post
title: "Adventures with PowerShell: Episode 1 – 64 vs 32 bit"
date: 2013-02-09 08:05:05 -0700
comments: false
categories: powershell
--- 
I have been working to learn/use more Powershell at work and as with any
new language/technology the learning process has been fraught with strange
gotchas and mysteries. For example, my friend had trouble with Powershell
stopping processing because _[he was clicking wrong](http://fitzse.wordpress.com/2013/02/04/powershell-console-select-mode/)_!

My most recent learning adventure was that the contents of certain system
directories may not be what they appear! I was goofing around with learning how
to install Powershell modules into the global location (making the module
accessible to all users). The typical advice you get on this matter is simply
to drop your module into ```C:\Windows\System32\WindowsPowerShell\v1.0\Modules``` and
you will be able to ```Import-Module``` to your hearts content.

This worked great for me until I tried to load my module with the x86 build of
PowerShell. The module could not be found.

Here is the file listing for the global module location from Windows Explorer.

![Windows Explorer system32 Listing](/images/blog/2013-02-09/folder.png)

And here is the file listing for the global module location as seen by the x64
PowerShell.

![x64 Powershell system32 Listing](/images/blog/2013-02-09/x64.png)

And here is the file listing for the global module location as seen by the x86
PowerShell.

![x86 Powershell system32 Listing](/images/blog/2013-02-09/x86.png)

Same file location, different contents. WAT?

After much poking around I finally figured out that when you are running the
x86 build of Powershell, the contents of
```C:\Windows\System32\WindowsPowerShell\v1.0\Modules``` are _actually_ the contents of
```C:\Windows\SysWOW64\WindowsPowerShell\v1.0\Modules```.

Unbeknownst to me at the time, the developers of PowerShell were actually
trying to help me out with this decision. Basically, the remapping of the file
system allows you to always use the same pathing without worrying about which
version of PowerShell you are running BUT allowing the system to segregate
modules that will only work in x64 environments. The tricky bit is that for
module authors that want their modules to be available to all versions of
PowerShell then they must install the module in _both_ locations. Unfortunately,
this behaviour does not seem to be well documented (or my Google-fu needs
work).

Good luck with your modules!
