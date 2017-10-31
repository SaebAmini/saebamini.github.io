---
layout: post
title: Authenticating Azure Service Management API requests with certificates
---

## Rant ##

Recently I needed to make a few simple calls to the Azure Management API from a web app; which turned out to be surprisingly much less straightforward than I expected it to be.

First, if you search for NuGet packages, you'll find a plethora of ambigously named ones. Some start with [`Microsoft.Azure.Management.*`](https://www.nuget.org/packages/Microsoft.Azure.Management.Compute), some with [`Microsoft.WindowsAzure.Management.*`](https://www.nuget.org/packages/Microsoft.WindowsAzure.Management.Compute/), both seem to be actively maintained with no clear documentation on what the difference is. There's also [some guides on MSDN](https://msdn.microsoft.com/en-us/library/dn722415.aspx) to help you get started but they are already outdated since a year ago, using obsolete methods that have been removed from newer library versions. Then if you research some more you come across setting up an Azure Active Directory application, client secrets, management certificates etc. with a lot of that information outdated due to the seemingly rapid changes Azure has gone through recently which has somehow become a cool thing nowadays as long as you're doing [Semantic Versioning](https://en.wikipedia.org/wiki/Software_versioning).

So to hopefully save you from the frustration I went through, I'll share the exact steps to a solution I finally arrived at that works (at least with the current API and library versions)


## Why certificates? ##

Although it's also possible to authenticate via the normal oAuth prompt flow, I'm not going to go into that approach here. That's because the Azure AD team [has removed Refresh Tokens](http://www.cloudidentity.com/blog/2015/08/13/adal-3-didnt-return-refresh-tokens-for-5-months-and-nobody-noticed/) from [ADAL](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-authentication-libraries) since 3.x.

What that means is the (Access) Token you end up with after going through the flow is only good for its 1 hour lifetime, and you can't manually refresh it using a Refresh Token. Well, that's not _completely_ accurate; the Refresh Token hasn't been really removed under the hood, it's just been _encapsulated_ from the interface, it will still remain in a `TokenCache` and the next time you want to acquire an access token, if the cache has a valid refresh token, it will be used to get a new access token silently rather than prompting the user. The problem is that cache is in-memory out of the box, so unless you pass in your own implementation of the TokenCache that uses persistent storage, you can't leverage the full life-time window of the Refresh Token, and even then, that'd be just 14 days (up to a maximum of 90 days if it's not left unused for any 14 days).

There's some talk of the lifetimes becoming configurable with the Refresh Tokens having a maximum life time of "Until Revoked" which you might be more familiar with when working with oAuth, but until that happens, and having to continuously prompt your users for authentication isn't acceptable in your situation, say a background app that has to continuously manage Azure resources, you're better off using [Management Certificates](https://docs.microsoft.com/en-us/azure/cloud-services/cloud-services-certs-create) which might be a bit more complex to set up initially, but the good thing is  it's a one-off process and it will remain valid until the certificate is removed from the subscription.


## Solution ##

First, you need a certificate to provide to Azure. This certificate can be self-signed so you can generate it yourself.