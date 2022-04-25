---
layout: post
title:  Using Azure AD for SSO in your Teams App (Part 1 - iFrames)
excerpt: Teams SSO!
author: jack
---

One of the main reasons why we started Hack and Chat was to centrally capture answers/insights we provide to partners, for commonly asked stuff,  so we can share it from a central place, to allow us to distribute it widely for future asks! So here is probably the number 1 thing we get asked for guidance and support on: How to implement effective authentication for a Teams Apps/Integration:

When working with partners, to help them build their Teams Apps/Integration, I would say that the number 1 ask is usually around how to do effective authentication. This usually results in partners spending a significant amount of time & energy implementing Azure AD SSO for their Teams app. Over the course of this multi-part blog series, I want to try and fast-track your understanding around how to do Azure AD SSO when you are bringing an iFrame into Teams, via either the Tab, Meeting Side-Panel or Task Module interfaces. Firstly lets cover the whys, before we move onto the whats and hows:

<br>

### Why should I use Azure AD SSO for my Teams App?
Here is a question I hear frequently from partners who are just about to start work on their Teams App/Integration:

>Why would I want to use Azure SSO for my app, when I already have my own authentication in place?

It's a fair question, but keep in mind that *all* users of Teams sign into the Teams client using Azure AD. Because of this it makes sense to make accessing your application, from inside Teams, as seemless as possible (meet users where they are). By implementing Teams SSO, it is possible to make it so that when a user opens your app in Teams, they are automatically provided with access to everything they expect. With the latest Teams SSO function in the Teams SDK, you can also make this work across devices, so you get SSO across the desktop Teams app, web Teams app and mobile Teams app!

This then removes a barrier to adoption, which should then result in increased usage/adoption of your Teams app. For most partners, usage is an important calculation that is used when determining the ROI of their Teams app, and is used to justify further time and money to spend on improving their initial Teams app release.

<br>

### What are the key concepts I need to understand to get started with solving any problems:
To help fast track your understanding around how Teams SSO works, here are the key things I think you need to understand:

**As we are specifically talking about Tab and Task Module interfaces to begin with... these are being delivered inside Teams as iFrames. Which means that Teams needs to control the SSO process:**
<br />
: It is likely that your web-app today contains some sort of protection on pages/routes/views, that require users to be authenticated to access the pages/routes/views, and subsequently make calls to your APIs to pull data and display it on those pages/routes/views. This protection probably checks if the user has a cookie, or an access token in local/session storage, and if the client doesn't, automatically redirects them to your login page. As Teams Tabs and Task Modules are iFrames, in most scenarios this redirect won't work. Your login page should have a content security policy that prevents it from being delivered inside an iFrame for security reasons (see [here](https://content-security-policy.com/?msclkid=f6754f1bc17711ecbe5a764f96d3960d) for more details on the types of attacks that can take place when using iFrames and why authentication providers block them). Because of this, firstly we need to detect if the app is being loaded inside Teams, and if so, stop the automatic redirect, and instead use the Teams SDK to control the auth process. To detect that the client is accessing your app from within Teams, you can use a mixture of query params and the Teams SDK (more on this later!). Once you have  detected if the client is within Teams, you would then use the Teams SDK to control the authentication process, and deliver it from outside of the iFrame (more on this later!). Once all this is in place, the Teams SDK should be able to get you an Azure AD issued Access Token, for the current logged in Teams user. This Access Token will have an audience of your application, and can be used to attempt to identify the user who is accessing your app from within Teams.
<br />
<br />

**Once you've got the Access Token from Azure AD, the next major challenge is implementing identity matching, to ensure that when your app is accessed inside Teams, the user is able to access the same resources as they would if they accessed your app outside Teams:**
<br />
: This is more challenging, and depends on what you are using for authentication/access. Your milage will vary depending on how easy it is for you to match a user based on their Azure AD claims, to a user in your system, and then the ability for you to generate a token from your authentication service, to provide back to the client to make calls to your back-end APIs. You can ask the user to sign-into your app once, to allow the ID matching to take place, if you can't automatically match them, but then all subsequence logins should be automatic, based on you matching them using the ID and TID claims in the Azure AD token (store that in your user store, alongside the user object for future ID matches). We'll cover the concepts and provide access to some sample code, to allow you to understand this in more detail in further blog posts, but for now, just know that identity matching needs to be implemented for this to work effectively.

### Disclaimer before we start looking at code, etc...
Most of the information (especially the technical stuff) shared in this blog post can be found scattered throughout various Microsoft Docs articles. The problem we find is that they lack context, and people tend to struggle to understand what's important and what isn't important in these docs articles. That being said, once you've read through this blog series, I'd recommend that you take a look at the following docs pages to give you further insight into how SSO can be implemented:

[Authentication in Teams Apps](https://github.com/Microsoft/botframework-sdk)

[Teams SSO](https://github.com/Microsoft/botframework-sdk)

[Teams SDK](https://github.com/Microsoft/botframework-sdk)

[Authentication Sample](https://github.com/Microsoft/botframework-sdk)

Please note that no official sample will demonstrate how you should do identity matching. I think we just expect your app/APIs to accept Azure AD issued access tokens... In our experience, this is almost never the case. To help with this, we have a sample app/lab available in Scott's GitHub repo: [AAD ID Matching Sample App Lab](https://github.com/Microsoft/botframework-sdk)


### How do I implement this then?
Okay... let's get started:
Firstly, in your Teams app manifest, I would strongly recommend adding a query string onto the end of your URL path. Something like this works well in most scenarios:

```json
"staticTabs": [
        {
            "entityId": "dashboard",
            "name": "Dashboard",
            "contentUrl": "https://www.appdomain.com/dashboard?inTeams=true",
            "websiteUrl": "https://www.appdomain.com/dashboard",
            "scopes": [
                "personal"
            ]
        },
]
```

The ```?inTeams=true``` can then be processed by your web-app to detect if this is likely to be loaded inside Teams, using something like:

```js
var urlParams = new URLSearchParams(window.location.search);
            var adminConsent = urlParams.get('isTeams');
            if (adminConsent == 'True') {
                inTeams = true;
            }
```

You will then need to call the Teams SDK if inTeams (the variable) is true. What you'll want to do is first make sure that the Teams SDK is available, 

Then, we need to write some logic into the page, to detect if that query param exists, and if so, get it to initalise the Teams SDK and get the context.
All being well, if the context is returned, we know that we are in Teams. So we can then start using the Teams SDK to control the authentication process:
There is a couple of amazing JavaScript functions in the Teams Client SDK that you need to understand:
getAuthToken()
&
authenticate.Authentication()

INSERT TEAMS SDK STUFF HERE FOR SSO


INSERT TEAMS SDK STUFF HERE FOR FALLBACK AUTH

Auth Start and Auth End page is key to success. 

This should then return to us an access token. We now need to *probably* get this changed for an access token that your APIs/controllers will accept. I want to cover this topic in the next blog

> _"Surely you just use the upn or ObjectID of the AAD user account?"_



Likewise , `_appSettings.MicrosoftAppPassword` (C#) and `process.env.MicrosoftAppPassword` (TypeScript) refer to a secret that has been created against your app registration.

_C#_
```cs
using Microsoft.Identity.Client;

private async Task<string> GetTokenForApp(string tenantId)
{
    var builder = ConfidentialClientApplicationBuilder.Create(_appSettings.MicrosoftAppId)
        .WithClientSecret(_appSettings.MicrosoftAppPassword)
        .WithTenantId(tenantId)
        .WithRedirectUri("msal" + _appSettings.MicrosoftAppId + "://auth");

    var client = builder.Build();

    var tokenBuilder = client.AcquireTokenForClient(new[] { "https://graph.microsoft.com/.default" });

    var result = await tokenBuilder.ExecuteAsync();

    return result.AccessToken;
}
```

_TypeScript_
```ts
import { ConfidentialClientApplication } from '@azure/msal-node';

public async getAppToken(tenantId: string) : Promise<string | undefined> {
    const cca = new ConfidentialClientApplication({
        auth: {
            clientId: process.env.MicrosoftAppId!,
            clientSecret: process.env.MicrosoftAppPassword,
            authority: `https://login.microsoftonline.com/${tenantId}`
        }
    });

    // Uses the client credential grant process to get a token
    const result = await cca.acquireTokenByClientCredential({
        scopes: ["https://graph.microsoft.com/.default"]
    });

    return result?.accessToken;
}
```



Happy coding!
