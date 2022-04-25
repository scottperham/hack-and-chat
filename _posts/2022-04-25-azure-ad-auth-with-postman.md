---
layout: post
title:  How to obtain an Azure AD delegated and app access token using Postman
tags: [Azure AD, API, Postman, Graph API]
excerpt_separator: <!--more-->
author: jack
---

When partners are first starting to work with the Microsoft Graph APIs, we usually send them over to the [Graph Explorer](https://developer.microsoft.com/en-us/graph/graph-explorer), so they can interact with the APIs in a safe space. The Graph Explorer provides you with a sandbox, where you can make calls under the context of a specific user (delegated), but it doesn't allow you to make calls to the Microsoft Graph APIs under the context of an app (deamon service).
Additionally, developers like to use Postman, or something similar, to craft these API calls, and then replicate these calls in code, using the Graph SDK, when they want to move it into their production app.
<!--more-->

To help with this, I've created a Postman collection that provides you with 2 requests, in a collection. These requests allow you to either get an Azure AD Access Token, using either the [Client Credentials Grant Flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow) (which provides a token under the context of an app) or the [Resource Owner Password Credentials Grant Flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth-ropc?msclkid=ce405a14c49f11ec8be142180afd07df) (which allows you to send a username and password via the APIs to Azure AD and it returns a token under the context of a user (delegated)).

:WARNING:
ROPC Grant Flow will not work for MFA enabled users and should not be used in production. It is designed to be used in test/dev scenarios, where an interactive user login, to obtain a delegated access token is not possible. For obtaining delegated access token in production scenarios, I'd recommend using the [Auth Code Grant Flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-auth-code-flow) instead.
:END:

[To get the Postman collection for Azure AD Auth, click here](https://github.com/OfficeDev/msteams-sample-contoso-hr-talent-app/blob/master/postman/Azure%20AD%20(Get%20Access%20Tokens).postman_collection.json) - you'll need to replace some of the variables, such as **tenantId** with the tenantId you are targetting.
Additionally, as this is all non-interactive, you will need to make sure that the scopes you need/request access to have been granted. This can be done via the Azure AD App Registration, or you can craft your own Azure AD Admin Consent URL, more details [here](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent#using-the-admin-consent-endpoint).

Remember that for app permissions, all the scopes you require must be listed in your Azure AD App Registration, as you can only work with /.default scope (which essentially looks at what is configured and consented to in your Azure AD App Registration). For delegated access tokens, you can request the scopes you want in your access token in the body of the request you are sending in Postman, they don't have to be explicitly listed in your Azure AD App Registration (although, best practice dictates that you want a superset of all the scopes your app requires listed in your app registration).

Assuming all works out as expected, when you make those calls to Azure AD via Postman, it will return an Access Token in the response, that you can use for subsequent calls to the Microsoft Graph API (using the authorization header, with this access token listed as the bearer token).

Cheers!

Jack