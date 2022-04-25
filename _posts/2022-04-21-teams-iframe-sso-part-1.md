---
layout: post
title:  Using Azure AD for SSO in your Teams App (Part 1 - iFrames)
tags: [Azure AD, JavaScript, Microsoft Teams]
excerpt_separator: <!--more-->
author: jack
---

One of the main reasons why we started Hack and Chat was to centrally capture answers/insights we provide to partners, for commonly asked stuff,  so we can share it from a central place, to allow us to distribute it widely for future asks! So here is probably the number 1 thing we get asked for guidance and support on: How to implement effective authentication for a Teams App/Integration:
<!--more-->

When working with partners, to help them build their Teams Apps/Integration, I would say that the number 1 ask is usually around how to do effective authentication. This usually results in partners spending a significant amount of time & energy implementing Azure AD SSO for their Teams app. Over the course of this multi-part blog series, I want to try and fast-track your understanding around how to do Azure AD SSO when you are bringing an iFrame into Teams, via either the Tab, Meeting Side-Panel or Task Module interfaces. Firstly lets cover the whys, before we move onto the whats and hows:

<br />

### Why should I use Azure AD SSO for my Teams App?
Here is a question I hear frequently from partners who are just about to start work on their Teams App/Integration:

>Why would I want to use Azure SSO for my app, when I already have my own authentication in place?

It's a fair question, but keep in mind that *all* users of Teams sign into the Teams client using Azure AD. Because of this it makes sense to make accessing your application, from inside Teams, as seemless as possible (meet users where they are!). By implementing Teams SSO, it is possible to make it so that when a user opens your app in Teams, they are automatically provided with access to everything they expect. With the latest Teams SSO function in the Teams SDK, you can also make this work across devices, so you get SSO across the desktop Teams app, web Teams app and mobile Teams app!

This then removes a barrier to adoption, which should then result in increased usage/adoption of your Teams app. For most partners, usage is an important calculation that is considered when determining the ROI of their Teams app, and is often used to justify further time and money to spend on improving their initial Teams app release.

<br />

### What are the key concepts I need to understand to get started with solving any problems:
To help fast track your understanding around how Teams SSO works, here are the key things I think you need to understand:

**As we are specifically talking about Tab and Task Module interfaces to begin with... these are being delivered inside Teams as iFrames. Which means that Teams needs to control the SSO process:**
<br />
: It is likely that your web-app today contains some sort of protection on pages/routes/views, that require users to be authenticated to access and subsequently make calls to your APIs to pull data and display it on them. This protection probably checks if the user has a cookie, or an access token in local/session storage, and if the client doesn't, automatically redirects them to your login page. As Teams Tabs, Meeting Side-Panels and Task Modules are iFrames, in most scenarios this redirect won't work. Your login page will/should have a content security policy that prevents it from being delivered inside an iFrame for security reasons (see [here](https://content-security-policy.com/?msclkid=f6754f1bc17711ecbe5a764f96d3960d) for more details on the types of attacks that can take place when using iFrames and why authentication providers block them). Because of this, firstly we need to detect if the app is being loaded inside Teams, and if so, stop this automatic redirect that the protection provides, and instead use the Teams SDK to control the auth process. To detect that the client is accessing your app from within Teams, you can use a mixture of query params and the Teams SDK. Once you have detected if the client is within Teams, you would then use the Teams SDK to control the authentication process, and deliver it from outside of the iFrame. When you have all of this in place, the Teams SDK should be able to get you an Azure AD issued Access Token, for the current logged in Teams user. This Access Token will have an audience of your application, and can be used to attempt to identify the user who is accessing your app from within Teams.
<br />
<br />

**Once you've got the Access Token from Azure AD, the next major challenge is implementing identity matching, to ensure that when your app is accessed inside Teams, the user is able to access the same resources as they would if they accessed your app outside Teams:**
<br />
: This is more challenging, and depends on what you are using for authentication/access. Your milage will vary depending on how easy it is for you to match a user based on their Azure AD claims, to a user in your system, and then the ability for you to generate a token/sessionID from your identity/access service, to provide back to the client to make calls to your back-end APIs. You can ask the user to sign-into your app once, to allow the ID matching to take place, if you can't automatically match them, but then all subsequence logins should be automatic, based on you matching them using the ID and TID claims in the Azure AD token (store that in your user store, alongside the user object for future ID matches). We'll cover the concepts and provide access to some sample code, to allow you to understand this in more detail in further blog posts, but for now, just know that identity matching needs to be implemented for this to work effectively.

### Disclaimer before we start looking at code, etc...
Most of the information (especially the technical stuff) shared in this blog post can be found scattered throughout various Microsoft Docs articles. The problem we find is that they lack context, and people get a bit lost in them... That being said, once you've read through this blog series, I'd recommend that you take a look at the following docs pages to give you further insight into how SSO can be implemented:

[Authentication in Teams Apps](https://docs.microsoft.com/en-us/microsoftteams/platform/concepts/authentication/authentication)

[Teams SSO](https://docs.microsoft.com/en-us/microsoftteams/platform/tabs/how-to/authentication/auth-aad-sso?tabs=dotnet)

[Authentication Sample](https://github.com/OfficeDev/Microsoft-Teams-Samples/tree/main/samples/tab-sso)

Please note that there are no official Microsoft samples that demonstrate how you should do identity matching. I think we just expect your app/APIs to accept Azure AD issued access tokens, although... In our experience, this is almost never the case. To help with this, we have a sample app/lab available in Scott's GitHub repo: [AAD ID Matching Sample App Lab](https://github.com/scottperham/AADLab)


### How do I implement this then?
Okay... let's get started:
Firstly, in your Teams app manifest, I would strongly recommend adding a query string onto the end of your URL paths, for the tabs in your Teams App. Something like this works well in most scenarios:

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

You will then need to call the Teams SDK if inTeams (the variable) is true. What you'll want to do is first make sure that the Teams SDK is available, details on how to make the script available can be found [here](https://docs.microsoft.com/en-us/microsoftteams/platform/tabs/how-to/using-teams-client-sdk)

Once that's all in order, you will then need to initialize the Teams SDK, based on the variable inTeams == true:

```js
if (inTeams == true)
    {
        microsoftTeams.initialize()       
    }
else
    {
        //do your normal auth flow (check for token/sessionId in local storage/cookies and if not then redirect to login page)
    }
```

This will then enable you to use the Teams JavaScript client SDK, to perform SSO, get context, request access to device media, etc...

Now, you need to setup your Azure AD App Registration, add the WebApplicationInfo properties to your Teams App Manifest and call the getAuthToken() function, that the Teams SDK provides, which if all is well will return an Azure AD Access Token. Details on how to do this can be found [here](https://docs.microsoft.com/en-us/microsoftteams/platform/tabs/how-to/authentication/auth-aad-sso?tabs=dotnet)

:WARNINGSTART:
Please follow the steps in the Microsoft Docs Teams SSO article carefully. From experience, we tend to see that most issues with the getAuthToken() function not returning an access token are related to some configuration issue that is present in the Azure AD App Registration. Typically because the API URI is malformed and doesn't match the standard detailed in the document, or the access_as_user scope is not setup correctly to allow the Teams to get an access token, for your app, with that scope included
:WARNINGEND:

Okay! So all being well, you should now have an access token that is returned when you call getAuthToken(). If you take this access token, and head over to https://jwt.ms/ you can decode the access token and take a look at the claims inside. These claims should include UPN/email (which is typically used to identity a user and match them to an existing user in the system), tenantId (TID), givenName and surname. You can also add additional optional claims into the token, by adjusting some of the settings in your Azure AD App Registration.

> This is all great when it works... but what happens if there is a consent/permissions issue, or just another issue in general? How do we handle fallback authentication for Teams apps, given that we can't just redirect to the login page?

I'm glad you asked! The way to handle fallback auth, is to catch an exception when getAuthToken() is called, and then utilise the authentication.authenticate() function, to pop out a modal window (which isn't an iFrame), that can be used to allow you to login to your identity service, and then pass a token or sessionId back down to the Teams session. The key to all of this is to craft an AuthEnd (and possibly an AuthStart) static view, to process the success or failure of an authentication, and then report that back to the Teams client. Firstly, here is the code that contains a function, that will open the AuthStart.html page inside the model authentication window:

```js
function teamsFallbackAuth() {

    microsoftTeams.authentication.authenticate({
        url: window.location.origin + "/StaticViews/AuthStart.html",
        width: 600,
        height: 535,
        successCallback: function (result) {
        },
        failureCallback: function (reason) {
            handleAuthError(reason);
        }
    });
}
```

Here is an example [AuthStart](https://github.com/jalew123/msteams-sample-contoso-hr-talent-app/blob/master/src/StaticViews/Login.html) page - as you can see, it's very simple. It's purpose is to setup MSAL and then redirect to Azure AD for authentication, with the correct parameters passed to the Azure AD service, so that it can perform the authentication and the redirect to the AuthEnd.html page if it's successful.

And here is an example of an [AuthEnd](https://github.com/jalew123/msteams-sample-contoso-hr-talent-app/blob/master/src/StaticViews/LoginResult.html) page - as you can see, this is also fairly straight forward. It initialised the Teams SDK, processes the access token using MSAL, and then uses the Teams SDK to report success (or failure), which passes the access token back down to the iFrame inside Teams and closes the modal window.

Most partners use the 'fallback' authentication approach to do the initial identity matching, if they can't automatically match a user on UPN/email claims inside the access token. They would then use the fallback auth to open their existing login page, ask the user to sign-in once, and then store the user's OID and TID alongside that user in their identity database for subsequent identity matches. 

Anyway, that's enough for this post. Next time, we'll explore how you can do the identity matching, and issue an access token (or SessionId) that your APIs will accept, based on the user having their Azure AD Access Token available through the getAuthToken() function. This will be more theoretical, as it's different for each app, depending on what they use to manage identities and access in their app.

Happy coding!
