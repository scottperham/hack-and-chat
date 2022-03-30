---
title:  "Proactive messaging in Teams"
---

## Proactive messaging in Teams

More and more of us are spending our days working in Microsoft Teams and with the [announcement that it's now reached a whopping 270 million monthly active users](https://twitter.com/fxshaw/status/1486107743320612867), we're seeing loads more high quality apps being published to the store... the most common feature we see of these apps is the ability to send a notification to a user.

There are a [fair few samples](https://github.com/microsoft/BotBuilder-Samples/tree/main/samples/csharp_dotnetcore/16.proactive-messages) out there that demonstrate how proactive messaging works but I still find that even though it's arguable the first feature people think of when they talk about integrating with Teams, it seems to be one of the main things people struggle to implement.

So... Here's how you do it!

Using the [Bot Framework SDK](https://github.com/Microsoft/botframework-sdk) you ultimately need to build a set of conversation parameters to be able to send a message. One of the properties of this conversation parameters object is a list of members to send the message to, however the tricky part here is getting the ID of the user.

> _"Surely you just use the upn or ObjectID of the AAD user account?"_

You'd think... but unfortunately for us, Teams uses specially encoded IDs internally to represent users, channels, apps, and pretty much anything else you might want to reference. Furthermore, there is no simple way to request these or build these yourself (except your bot ID, but more on that later!).

This really only leaves us with 2 options...

### **Option 1**: Cache the conversation reference when a user is added to the conversation

[This is how the samples do it](https://github.com/microsoft/BotBuilder-Samples/tree/main/samples/csharp_dotnetcore/16.proactive-messages) and it works great... if you can guarantee perstistant storage of these references. If you can't justify the operational overhead, you lose them for any reason or proactive messaging is added _after_ a chat feature already exists in your app, you're sort of stuck.

>Wouldn't it be better though if you could programmatically get this conversation reference on demand? 

Yes... yes it would, and that brings me onto option 2.

### **Option 2**: Programmatically get this conversation reference on demand

The first thing to note is that this requires application scopes, yep, the dreaded ADMIN CONSENT... the upside though is that these are probably the least-scary admin scopes you'll come across!

* **TeamsAppInstallation.ReadWriteSelfForUser.All** - 
This scope allows you to proactively install your app for a given user and query the installation status. Importantly the "self" bit is what makes this scope rather less scary as it only allows your app to manage your app. When you query the installed apps for a given user using a token with this scope, it will only return a reference to your app, regardless of what else the user has installed. Likewise, it will only allow you to install your app on behalf of a user and not anything else.

* **AppCatalog.Read.All** - 
What if the user doesn't have the app installed? Well, this scope will let you search the Teams app store and the organisation store to find it. You can then use the previous scope to proactively install it for the user.

Now that we have that bit out of the way, here is the process for programmatically getting the conversation reference needed to send a message to a user:

These steps make reference to some configuration values

### 1. Get an app token

First we need to get our application token. There isn't anything clever in this, it just uses the MSAL library to and the [client credentials grant flow](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow).

In this example, `_appSettings.MicrosoftAppId` (C#) and `process.env.MicrosoftAppId` (TypeScript) are references to the Client ID of your AAD app registration. (If you're unsure of what that is, you might want to [start here](https://docs.microsoft.com/en-us/powerapps/developer/data-platform/walkthrough-register-app-azure-active-directory#:~:text=%20Create%20an%20application%20registration%20%201%20Sign,your%20application%27s%20registration%20information%3AIn%20the%20Name...%20See%20More.) instead).

Likewise , `_appSettings.MicrosoftAppPassword` (C#) and `process.env.MicrosoftAppPassword` (TypeScript) refer to a secret that has been created against your app registration.

_C#_
```C#
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
```TypeScript
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

### 2. Determine whether the app is currently installed for the user

Now we need to call the installed apps Graph API endpoint to determine whether the app has been installed for the given user.

To do this we use the [users/{upn}/teamwork/installedApps](https://docs.microsoft.com/en-us/graph/api/userteamwork-list-installedapps?view=graph-rest-1.0&tabs=http) Graph API call.

If the app cannot be found, this is most likely because the user doesn't have it installed (we can fix this!)

In this example, `_appSettings.TeamsAppId` (C#) and `process.env.TeamsAppId` (TypeScript) refer to the ID (GUID) inside the Teams app manifest.

_C#_
```C#
var installedApps = await graphClient.Users[upn].Teamwork.InstalledApps
    .Request()
    .Filter($"teamsApp/externalId eq '{_appSettings.TeamsAppId}'")
    .Expand("teamsApp")
    .GetAsync(cancellationToken);

var app = installedApps.FirstOrDefault();
if (app == null)
{
    // App not installed for user
}
else {
    // This is the internal installation ID of your app in the target tenant
    var installationId = app.Id; 
    ...
}
```

_TypeScript_
```TypeScript
const installedApps = await graphClient.api(`/users/${upnOrOid}/teamwork/installedapps`)
    .filter(`teamsApp/externalId eq '${process.env.TeamsAppId}'`)
    .expand("teamsApp")
    .get() as ODataCollection<UserScopeTeamsAppInstallation>;

if (installedApps.value.length == 0) {
    // App not installed for user
}
else {
    // This is the internal installation ID of your app in the target tenant
    const installationId = installedApps.value[0].id;
    ...
}
```

### 3. Install the app (if necessary)

Find our application in the app store or organisation store using [appCatalogs/teamsApps](https://docs.microsoft.com/en-us/graph/api/appcatalogs-list-teamsapps?view=graph-rest-1.0&tabs=http) and then install it with [users/{upn}/teamwork/installedApps](https://docs.microsoft.com/en-us/graph/api/userteamwork-post-installedapps?view=graph-rest-1.0&tabs=http)

_C#_
```C#
var teamsApps = await graphClient.AppCatalogs.TeamsApps
    .Request()
    .Filter($"distributionMethod eq 'organization' and externalId eq '{_appSettings.TeamsAppId}'")
    .GetAsync(cancellationToken);

var teamApp = teamsApps.FirstOrDefault();

if (teamApp != null)
{
    try
    {
        var installBotRequest = new BaseRequest($"https://graph.microsoft.com/v1.0/users/{upn}/teamwork/installedApps", graphClient)
        {
            Method = HttpMethods.POST,
            ContentType = MediaTypeNames.Application.Json
        };

        await installBotRequest.SendAsync(
            new TeamsAppInstallation
            {
                AdditionalData = new Dictionary<string, object>
                {
                    { "teamsApp@odata.bind", $"https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/{teamApp.Id}" }
                }
            }, cancellationToken);
    }
    catch (ServiceException svcEx) when (svcEx.StatusCode == System.Net.HttpStatusCode.Conflict)
    {
        // Conflict means the app was already installed
    }
}
```

_TypeScript_
```TypeScript
const teamsApps = await graphClient.api("/appCatalogs/teamsApps")
    .filter(`externalId eq '${process.env.TeamsAppId}'`)
    .get() as ODataCollection<any>;

if (teamsApps.value.length > 0 && teamsApps.value[0].id) {
    try {
        await graphClient.api(`/users/${upnOrOid}/teamwork/installedApps`).post({
            "teamsApp@odata.bind": `https://graph.microsoft.com/v1.0/appCatalogs/teamsApps/${installationId}`
        });
    }
    catch (err: any) {
        if (!err.hasOwnProperty("statusCode") || err["statusCode"] !== 409) {
            throw err;
        }

        // Conflict means the app was already installed
    }

    ...
}
```

### 4. Get the internal user reference

To get the internal user reference (also known as a ChannelAccount) we call the [chat graph endpoint for the installed app](https://docs.microsoft.com/en-us/graph/api/userscopeteamsappinstallation-get-chat?view=graph-rest-1.0&tabs=http)

_C#_
```C#
var chat = await graphClient.Users[upn].Teamwork.InstalledApps[installationId].Chat
    .Request()
    .GetAsync(cancellationToken);

var credentials = new MicrosoftAppCredentials(_appSettings.MicrosoftAppId, _appSettings.MicrosoftAppPassword);

var connectorClient = new ConnectorClient(new Uri(_appSettings.ServiceUrl), credentials);

var members = await connectorClient.Conversations.GetConversationMembersAsync(chatId);

var internalUserObject = members[0];
```

In the case of TypeScript / Javascript, there isn't a built-in ConnectorClient class in the framework, but we can create our own! [Here is one I created](https://github.com/scottperham/hr-talent-node/blob/master/src/services/data/botService.ts) that exposes the [v3/conversation/members bot framework api endpoint](https://docs.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-connector-api-reference?view=azure-bot-service-4.0#get-conversation-members).

_TypeScript_
```TypeScript
const chat = await graphClient
    .api(`/users/${upnOrOid}/teamwork/installedApps/${installationId}/chat`)
    .get() as Chat;

const connectorClient = new ConnectorClient();
const members = await connectorClient
                    .getConversationMembers(process.env.ServiceUrl!, chatIdResponse.chatId!);

var internalUserObject = members[0];
```

>**Note:** This has a side effect of calling your bot with a ConversationUpdate activity as if new members have been added to the chat.

### 5. Create the conversation parameters

Although we weren't able to craft or request the internal user id, we can programmatically create the bot id! This is in the following format "28:{_Your AAD client ID_}"

_C#_
```C#
var conversationParameters = new ConversationParameters
{
    IsGroup = false,
    Bot = new ChannelAccount
    {
        Id = "28:" + credentials.MicrosoftAppId
    },
    Members = new ChannelAccount[] { members[0] },
    TenantId = tenantId,
};
```

_TypeScript_
```TypeScript
const conversationParameters: Partial<ConversationParameters> = {
    isGroup: false,
    bot: {
        id: `28:${process.env.MicrosoftAppId}`,
        name: ""
    },
    tenantId,
    members: [internalUserObject]
};
```

### 6. Send the activity

Finally, we can weave our way around the confusing set of nested callbacks to craft the TurnContext we need to send the activity!

We have a new configuration value here, `_appSettings.ServiceUrl` (C#) and `process.env.ServiceUrl` (TypeScript). This is basically the base url for bot service and will be https://smba.trafficmanager.net/{your region} - now, {your region} can be a few possible values... I typically fallback to "uk" but you might want to consider overriding this with the service url that appears in any activity that is sent to your bot as this will ensure that you have the correct region. (Having said that, my tests seem to confirm that as long as it's a valid region value, the messages will still be routed!)

_C#_
```C#
await ((CloudAdapter)_adapter).CreateConversationAsync(credentials.MicrosoftAppId, null, _appSettings.ServiceUrl, credentials.OAuthScope, conversationParameters, async (t1, c1) =>
{
    var conversationReference = t1.Activity.GetConversationReference();
    await ((CloudAdapter)_adapter).ContinueConversationAsync(credentials.MicrosoftAppId, conversationReference, async (t2, c2) =>
    {
        await t2.SendActivityAsync(activity, c2);
    }, cancellationToken);
}, cancellationToken);
```

_TypeScript_
```TypeScript
await this.adapter.createConversationAsync(process.env.MicrosoftAppId!, "", process.env.ServiceUrl!, "https://api.botframework.com", <ConversationParameters>conversationParameters, async (t1) => {
    const conversationReference: ConversationReference = {
        activityId: t1.activity.id,
        user: t1.activity.from,
        bot: t1.activity.recipient,
        conversation: t1.activity.conversation,
        channelId: t1.activity.channelId,
        locale: t1.activity.locale,
        serviceUrl: t1.activity.serviceUrl
    }
    await this.adapter.continueConversationAsync(process.env.MicrosoftAppId!, conversationReference, async (t2) => {
        await t2.sendActivity(activity);
    });
});
```

Phew! That most definitely feels more complex than it needs to be, but until the bot service supports and endpoint to retrieve those internal user Ids, this is, unfortunately the way it has to work!

A fully functional sample that uses this approach can be found [here for C#](https://github.com/OfficeDev/msteams-sample-contoso-hr-talent-app) and here for [Node/TypeScript](https://github.com/scottperham/hr-talent-node)

Happy coding!
