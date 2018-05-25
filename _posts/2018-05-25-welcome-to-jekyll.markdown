---
layout: post
title:  "Xamarin Authentication as existing user via Salesforce Oauth"
date:   2018-05-25 09:00:54 +0300
categories: [Xamarin, iOS, Android]
tags: [Xamarin, iOS, Android]
---

Hello, here I am going to share high level approach of implementation Oatuh authentication via [https://www.salesforce.com/](Salesforce) (next SF) at xamarin app (iOS / Android).
Interesting in this task is the case when you have user's accounts in your app already and you need to let users login into app through theirs Salesforce account.

SF provided a Xamarin component called [https://components.xamarin.com/view/SalesforceSDK](SalesforceSDK 1.4.4.0) with source code on [https://github.com/xamarin/SalesforceSDK](https://github.com/xamarin/SalesforceSDK) however last published data of component September 29 of 2015 which is quite old and looks like yet another obsolete SDK.


`Official documentation provides an example of usage a component`:
{% highlight C# %}
// Creates our connection to salesforce.
var client = new SalesforceClient(clientId, clientSecret, redirectUrl);

// Get authenticated users from the local keystore
var users = client.LoadUsers();
if (!users.Any())
{
        client.AuthenticationComplete += (sender, e) => OnAuthenticationCompleted(e);
        // Starts the Salesforce login process.
        var loginUI = client.GetLoginInterface();
        DisplayThe(loginUI);
}
else
{
        // We're ready to fetch some data!
        // Let's grab some sales accounts to display.
        IEnumerable<SObject> results =
                    await client.ReadAsync("SELECT Name, AccountNumber FROM Account");
 
        DoSomethingAmazingWith(results);
}
{% endhighlight %}

And guess what? That won't work no metter how I tried.


Main problem is that `OnAuthenticationCompleted` handler was never triggered. Talking a little bit ahead  I may assume that it happens because of SF changed authentication flow on theirs web app and make component not compatible.

So I decided to make some hack.