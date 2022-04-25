---
layout: post
title:  Adaptive card templates
tags: [Adaptive Cards, Templating, C#, JavaScript]
excerpt_separator: <!--more-->
author: scott
---

[Adaptive cards](https://adaptivecards.io/) are cool! Rich, platform-agnostic UI snippets defined by a little JSON... what's not to love?

How about making them even cooler with some fancy runtime data binding?

<!--more-->

When you need to build data-driven cards there are a couple of options open to you. The first is to simply define the JSON for the final card dynamically. 

Although this is made slightly easier with the Adaptive Card SDKs that are available for [.NET](https://docs.microsoft.com/en-us/adaptive-cards/sdk/authoring-cards/net) and [JavaScript](https://docs.microsoft.com/en-us/adaptive-cards/sdk/authoring-cards/javascript), the downside to this approach is that it tends to require a lot of declaritive code and it's not that easy to take a beautifully designed card from the [Adaptive Card designer](https://adaptivecards.io/designer/) and re-create that with the available object models.

The second way of handling this (and in my opinion, the better way!) is to use **_templating_**.

Using this approach means you can still use tools like the [Adaptive Card designer](https://adaptivecards.io/designer/) or the [Adaptive Card Studio extension for VSCode](https://marketplace.visualstudio.com/items?itemName=madewithcardsio.adaptivecardsstudiobeta) for that great WYSIWYG experience _and_ get the benefit of dynamic data-driven content.

## Here's how it works!

If you've used the Adaptive Card designer before, you've likely already been exposed to the basics of templating - a value in the card JSON surrounded by `${...}` which represents the binding. In the screenshot below there are two of these, `${title}` and `${description}`.

![Adaptive Card designer](/assets/adaptive-card-templates/designer.png)

> This is great in the designer, but how do I do this in code?!

Great question!

## Using Node / Javascript:

The functionality is exposed in the handy dandy NPM package, "adaptivecards-templating".

This can be installed with the following command line:

```
npm install adaptivecards-templating
```

Now all you need to do is load and "expand" the template!

```js
import * as act from 'adaptivecards-templating';

// Note the escaped $ in the ES6 template literal...
const adaptiveCardJson = JSON.parse(`{
    "type": "AdaptiveCard",
    "body": [
        {
            "type": "TextBlock",
            "size": "medium",
            "weight": "bolder",
            "text": "\${title}",
            "wrap": true
        },
        {
            "type": "TextBlock",
            "text": "\${description}",
            "wrap": true
        }
    ],
    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
    "version": "1.3"
}`);

const template = new act.Template(adaptiveCardJson);
const payload = template.expand({
    $root: {
        title: "My dynamic title",
        description: "My dynamic description"
    }
});
```

Equally, you can achieve the same result with an object literal... because javascipt is cool like that!

```js
import * as act from 'adaptivecards-templating';

const adaptiveCardJson = {
    type: "AdaptiveCard",
    body: [
        {
            type: "TextBlock",
            size: "medium",
            weight: "bolder",
            text: "${title}",
            wrap: true
        },
        {
            type: "TextBlock",
            text: "${description}",
            wrap: true
        }
    ],
    $schema: "http://adaptivecards.io/schemas/adaptive-card.json",
    version: "1.3"
};

const template = new act.Template(adaptiveCardJson);
const payload = template.expand({
    $root: {
        title: "My dynamic title",
        description: "My dynamic description"
    }
});
```

Or, perhaps you would like to do something crazy like actually have your adaptive cards source controlled seperately from the code (it will never catch on!)?

```js
import * as fs from 'fs';
import * as act from 'adaptivecards-templating';

const adaptiveCardJson = JSON.parse(fs.readFileSync(/* Path */));

const template = new act.Template(adaptiveCardJson);
const payload = template.expand({
    $root: {
        title: "My dynamic title",
        description: "My dynamic description"
    }
});
```

Whichever way you do it, the value of `paylod` is the fully expanded adaptive card json (if you are using TypeScript, this will be of type `any`)

Now, depending on what you actually want to do with your expanded adaptive card, you might need to process it further - perhaps convert it back to a string:

```js
const payloadString = JSON.stringify(payload);
```

Or use [BotFramework SDK](https://github.com/microsoft/botframework-sdk) to create an adaptive card object:

```js
import { CardFactory } from 'botbuilder';

const card = CardFactory.adaptiveCard(payload);
```

## Using .NET / C#

In the case of .NET, there is also a neat little nuget package to get the same functionality:

_**NOTE:** At the time of writing this there is a strange version conflict in the Newtonsoft.Json package that's referenced by the templating package and a dependant one... unless you have already explicitly included Newtonsoft.Json in your project, the best way to get around this is to manually install this package first_

```
Install-Package Newtonsoft.Json
Install-Package AdaptiveCards.Templating
```

And the usage is much the same as above:

```cs
using AdaptiveCards.Templating;

var templateJson = @"
{
    ""type"": ""AdaptiveCard"",
    ""body"": [
        {
            ""type"": ""TextBlock"",
            ""size"": ""medium"",
            ""weight"": ""bolder"",
            ""text"": ""${title}"",
            ""wrap"": true
        },
        {
            ""type"": ""TextBlock"",
            ""text"": ""${description}"",
            ""wrap"": true
        }
    ],
    ""$schema"": ""http://adaptivecards.io/schemas/adaptive-card.json"",
    ""version"": ""1.3""
}";

var template = new AdaptiveCardTemplate(templateJson);

var payload = template.Expand(new 
{
    title = "My dynamic title",
    description = "My dynamic description"
});
```

Here `payload` is a `string` representation of the resulting JSON.

Whichever language you use and whichever approach you decide to take, you've quickly and easily created a data-driven adaptive card!

## But wait, there's more!

You can do so much (_SO MUCH!_) with the [expression language behind adaptive card templates](https://docs.microsoft.com/en-us/adaptive-cards/templating/language#:~:text=Adaptive%20Card%20Templating%20is%20built%20on%20top%20of,just%20a%20small%20sampling%20of%20the%20built-in%20functions.)... definitely worthy of it's own post (spoiler). But to get you started, the [HR Talent App Sample for Teams](https://github.com/OfficeDev/msteams-sample-contoso-hr-talent-app-node) project makes extensive use of card templating - it even uses some of the other cool features like conditionals and itterative binding!

Happy coding!
