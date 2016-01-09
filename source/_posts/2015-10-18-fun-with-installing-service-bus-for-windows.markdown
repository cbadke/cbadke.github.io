---
layout: post
title: "Fun with Installing Service Bus for Windows"
date: 2015-10-18 07:40:14 -0600
author: Curtis Badke
comments: false
categories:
---
[tl;dr](#conclusion)

I recently started a toy project that will involve using Azure Service Bus to
communicate between multiple Azure WebJobs and Cloud Services. I'm not a fan of
doing development against actual live cloud services so I figured I would
install Service Bus 1.1 for Windows Server on my local dev machine to do my
development and then deploy a full instance of the application to Azure for
cloud testing.

I don't like developing against live cloud services because this tends to cause
collisions among team members (multiple people start posting events for example)
and there is a risk of publishing cloud credentials into public repositories.

This post will lay out a bunch of roadblocks I ran into with getting Service Bus
up and running on my local dev machine.

My machine:

  * Windows 10 installed with AAD user
  * VS2015
  * Service Bus 1.1

## <a name="installation"></a>Installation

Installing Service Bus itself is easy. Just use the Microsoft Web Platform
Installer. Search for 'Service Bus' and install 'Windows Azure Pack: Service Bus 1.1'.

![Search for Service Bus in Platform Installer](/images/blog/2015-10-18/search-results.png)

## Farm Creation

Configuration is where I ran into my first problems. My organization uses
Azure AD as a mirror of our main domain AD. Windows 10 gives the option during
installation to just login with your AAD account. This is very convenient but
leaves your computer in a strange state.

When you use AAD to create your user account your machine isn't actually connected
to your domain. You have a Workgroup machine but are logging in with what _feels_
like a domain account.

Service Bus did not react well to this situation. When creating a new farm using
`New-SBFarm` I would receive the error:

> New-SBFarm : Unable to cast object of type 'System.DirectoryServices.AccountManagement.GroupPrincipal' to type 'System.DirectoryServices.AccountManagement.UserPrincipal'.

This error seems to be related to the mismatch of my user (AAD/Domain) to my
machine which is tied to a Workgroup. According to the [System Requirements](https://msdn.microsoft.com/en-us/library/dn441409.aspx),
my options were to create a local admin account OR join my machine to a domain
and log in with a domain user with admin rights. I opted to add a local user and
farm creation worked fine.

I also found that I was not able to grant any sort of permissions to my
Service Bus farm and namespace to my AAD user. If you're on a workgroup you'll
need to use local users for pretty much all interactions with the Bus.

## <a name="running"></a>Starting Service Bus

In order to actually get Service Bus up and running the next step is to add
your local machine to the farm using `New-SBHost`. During this step I found that
my Service Bus Gateway would never actually start. Digging into the Service Bus
logs in the Event Viewer (Applications and Local Services -> Microsoft -> Microsoft-ServiceBus -> Operational)
I found an exception that the assembly Microsoft.Cloud.Common.AzureStorage could
not be loaded.

> Service Bus Gateway service failed to start, retry count 3.
> Exception message: An error occurred creating the configuration section handler for namespacePolicyDataStoreFactory:
> Could not load file or assembly 'Microsoft.Cloud.Common.AzureStorage, Version=2.1.0.0, Culture=neutral, PublicKeyToken=4fe77f22fa8374f3' or one of its dependencies.

After some searching I stumbled upon this [GitHub repo](https://github.com/matthewcanty/Microsoft.Cloud.Common.AzureStorage.FAKE.dll).
Apparently, if you install VS2015 you will run into a missing DLL problem because
Service Bus for Windows does not ship with this assembly. Normally this isn't a
problem but a change to mscorlib now causes the assembly to be loaded even though
the code is not actually used.

The solution is to install a fake version of the dll which contains the types
that are being loaded. I'm not terribly excited about registering some random
person's dlls into my system but I disassembled the fake assembly and it truly
does created a single empty class to get around a type reference issue. Because
the type isn't actually _used_ the lack of implementation is not an issue.

I downloaded and installed the dll as described in the repo and Service Bus
services were up and running.

## <a name="hello-world"></a>Hello World?

Service Bus is running, you've created a namespace, we should be able to try getting
two processes passing messages, right? Well, maybe.

It was during my 'hello world' testing that I ran into the most concerning of
the issues with setting up a local Service Bus as a dev time replacement for Azure.

I followed the typical Service Bus examples and was met with an exception:


> The remote server returned an error: (400) Bad Request. The api-version in the query string is not supported. Either remove it from the Uri or use one of '201 2-03,2012-08,2013-04,2013-07'

When you communicate with Service Bus you will typically install the Nuget
package WindowsAzure.ServiceBus. Service Bus uses a numbered API version that
is verified on connection. It seems Service Bus for Windows 1.1 is pretty out of
date compared to what is available in Azure proper. I was forced to downgrade
the WindowsAzure.ServiceBus package all the way back to 2.1.4.0 before I could
establish a connection with my local Bus.

I'm not sure what the ramifications will be to using such an old version of the
Service Bus API when it comes time to deploy my app into Azure. Documentation
says the Azure instance supports all the client versions back to the beginning 
of time so presumably if I'm not trying to use any of the fancy new features I
should be okay but time will tell.

I look forward to when Microsoft publishes a new on-prem Service Bus that more
closely matches what is actually running in Azure.

## <a name="certificates"></a>Certificate Fun

If you got as far as getting Service Bus running and installed the right client
libraries and were met with TLS/SSL security errors you've probably run into
an issue of certificates.

> System.UnauthorizedAccessException was unhandled
> Message: An unhandled exception of type 'System.UnauthorizedAccessException' occurred in Microsoft.ServiceBus.dll
> Additional information: The token provider was unable to provide a security token while accessing 'https://localhost:9355/yournamespace/$STS/Windows/'. Token provider returned message: 'The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel.'.

When you add your machine as a Service Bus host with `New-SBHost`, the
configuration script will generate new SSL certificates for your machine. These
certificates will be tied to your hostname and if you are on a domain I think
they are actually made for you fully qualified domain name. This means that you
cannot use other names (like localhost) to connect to your Service Bus. The
certificate names won't match and the trust connection will fail.

## <a name="conclusion"></a>Conclusion

Running Service Bus locally for dev is possible but not without configuration
annoyances. I haven't used it enough to know if there are any actual draw backs
once you have it working and it comes time to deploy to Azure. Perhaps another
post will provide some followup.

Things you'll need to be aware of for local Service Bus installation:

* If you are in a workgroup you must use local users, if you are in a
  domain you must use domain users. If you are on Windows 10 with an AAD user
  your machine is probably in a workgroup. [reference](https://msdn.microsoft.com/en-us/library/dn441409.aspx)
* If you have VS 2015, you need to install a [fake Microsoft.Cloud.Common.AzureStorage](https://github.com/matthewcanty/Microsoft.Cloud.Common.AzureStorage.FAKE.dll)
  assembly.
* You must use Nuget package WindowsAzure.ServiceBus 2.1.4.0 or _older_.
* You must address your Service Bus connections using your full machine name
  not a short name or something like localhost

Hopefully this saves someone hours of frustration.
