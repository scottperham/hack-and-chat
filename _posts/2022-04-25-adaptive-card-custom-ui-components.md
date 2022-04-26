---
layout: post
title:  Adaptive card custom UI
tags: [Adaptive Cards, C#]
excerpt_separator: <!--more-->
author: scott
---

There is a lot you can do to make [Adaptive cards](https://adaptivecards.io/) look great with what's available out of the box. There are scenarios though where you want that extra zing to make it pop, unfortunately you are only limited to the UI elements that are defined in the [JSON schema](https://adaptivecards.io/explorer/), but there are some things you can do to make your cards truly unique!

One approach is to build images dynamically!

<!--more-->

Did I say "one approach"... actually, this might be the only approach!

Take the following example:

![Adaptive Card](/assets/adaptive-card-custom-ui/card-basic.png)

This card represents a notification that a new blog post is available. To show the list of tags associated with the post, here we are using a [FactSet](https://adaptivecards.io/explorer/FactSet.html) and it does the job pretty well.

But it _could_ look like this:

![Adaptive Card](/assets/adaptive-card-custom-ui/card-with-badges.png)

_{Waits for gasps of astonishment...}_

Ok, I know this is a pretty trivial example, but it demonstrates the ability to put elements on the card that you can't define with the JSON alone.

And here's how it's done...

The process is really quite simple, you host an endpoint that essentially pretends to support a GET request for a static image but actually dynamically generates the image on each request - this bit is the easiest part, we just change the `Content-Type` response header to some image mime type, in this case `image/png`:

```cs
[Route("/images/badge")]
public async Task<IActionResult> GetBadge()
{
    Response.ContentType = "image/png";

    // Do stuff to build the image here...

    return Ok();
}
```

Now for the more complex part... drawing the image!

How you do this will very much depend on which graphics library you go with... in the not-so-olden days we'd use GDI+ and a combination of `Image`, `Pen`, `Brush` and `Path` types to draw the content to a `Graphics` context.

Something to mention here is that in recent years dotnet has become much more Linux friendly which is a good thing! But something it still lacks to this day is a really good support for cross-platform graphics.

I tend to host services on a combination of Linux and Windows so I've gone for a cross-platform library called [ImageSharp](https://sixlabors.com/products/imagesharp/). (All managed code, very cool library!)

First, let's install the nuget package:

```
Install-Package SixLabors.ImageSharp.Drawing -PreRelease
```

:WARNING:
-PreRelease is required here because currently the Drawing package is in beta!
:END:

Then, write some code:

```cs
[Route("/images/badge")]
public async Task<IActionResult> GetBadge()
{
    var fontCollection = new FontCollection();
    fontCollection.AddSystemFonts();

    var fontFamily = fontCollection.Get("Arial");
    var font = fontFamily.CreateFont(12, FontStyle.Regular);

    var options = new TextOptions(font);

    var fontRect = TextMeasurer.Measure("My Badge", options);

    using var img = new Image<Rgba32>((int)fontRect.Width + 20, (int)fontRect.Height + 10);

    img.Mutate(x => x.Fill(Color.Transparent));
    img.Mutate(x => ApplyRoundedCorners(x.Fill(Color.Orange).DrawText("My Badge", font, Color.White, new PointF(badge.PaddingX, badge.PaddingY)), 10));

    Response.ContentType = "image/png";

    // Here we save the image to the response stream
    await img.SaveAsPngAsync(Response.Body);

    return Ok();
}

private IPathCollection BuildCorners(int imageWidth, int imageHeight, float cornerRadius)
{
    // first create a square
    var rect = new RectangularPolygon(-0.5f, -0.5f, cornerRadius, cornerRadius);

    // then cut out of the square a circle so we are left with a corner
    IPath cornerTopLeft = rect.Clip(new EllipsePolygon(cornerRadius - 0.5f, cornerRadius - 0.5f, cornerRadius));

    // corner is now a corner shape positions top left
    //lets make 3 more positioned correctly, we can do that by translating the original around the center of the image

    float rightPos = imageWidth - cornerTopLeft.Bounds.Width + 1;
    float bottomPos = imageHeight - cornerTopLeft.Bounds.Height + 1;

    // move it across the width of the image - the width of the shape
    IPath cornerTopRight = cornerTopLeft.RotateDegree(90).Translate(rightPos, 0);
    IPath cornerBottomLeft = cornerTopLeft.RotateDegree(-90).Translate(0, bottomPos);
    IPath cornerBottomRight = cornerTopLeft.RotateDegree(180).Translate(rightPos, bottomPos);

    return new PathCollection(cornerTopLeft, cornerBottomLeft, cornerTopRight, cornerBottomRight);
}

// This method can be seen as an inline implementation of an `IImageProcessor`:
// (The combination of `IImageOperations.Apply()` + this could be replaced with an `IImageProcessor`)
private IImageProcessingContext ApplyRoundedCorners(IImageProcessingContext ctx, float cornerRadius)
{
    Size size = ctx.GetCurrentSize();
    IPathCollection corners = BuildCorners(size.Width, size.Height, cornerRadius);

    ctx.SetGraphicsOptions(new GraphicsOptions()
    {
        Antialias = true,
        AlphaCompositionMode = PixelAlphaCompositionMode.DestOut // enforces that any part of this shape that has color is punched out of the background
    });

    // mutating in here as we already have a cloned original
    // use any color (not Transparent), so the corners will be clipped
    foreach (var c in corners)
    {
        ctx = ctx.Fill(Color.Red, c);
    }
    return ctx;
}
```

There is a fair bit of code here but most of it comes from one of [ImageSharp's samples on GitHub](https://github.com/SixLabors/Samples/tree/main/ImageSharp/AvatarWithRoundedCorner).

Let's navigate to the endpoint and see the fruits of our labour...

```
<BASE URI>/images/badge
```

You should see something similar to the below:

![Badge](/assets/adaptive-card-custom-ui/browser-no-args.png)

This is cool... but we can make it a bit better by using some parameters to control how the badge is rendered.

Let's add a new type to hold that request data (from the querystring)

```cs
public class BadgeQuery
{
    public string Text { get; set; } = "Badge";
    public string Font { get; set; } = "Arial";
    public int TextHeight { get; set; } = 12;
    public int PaddingX { get; set; } = 10;
    public int PaddingY { get; set; } = 10;
    public int CornerRadius { get; set; } = 10;
    public string BackgroundColor { get; set; } = "Orange";
    public string TextColor { get; set; } = "Black";
    public string FontStyle { get; set; } = "Regular";
}
```

Now, we can update our code to use these values instead of the hard-coded ones...

```cs
[Route("/images/badge")]
public async Task<IActionResult> GetBadge([FromQuery] BadgeQuery badge)
{
    var fontCollection = new FontCollection();
    fontCollection.AddSystemFonts();

    var fontStyle = Enum.Parse<FontStyle>(badge.FontStyle);

    var fontFamily = fontCollection.Get(badge.Font);
    var font = fontFamily.CreateFont(badge.TextHeight, fontStyle);

    var options = new TextOptions(font);

    var fontRect = TextMeasurer.Measure(badge.Text, options);

    using var img = new Image<Rgba32>((int)fontRect.Width + badge.PaddingX * 2, (int)fontRect.Height + badge.PaddingY * 2);

    var backgroundColor = Color.Orange Color.Parse(badge.BackgroundColor);
    var textColor = Color.Parse(badge.TextColor);

    img.Mutate(x => x.Fill(Color.Transparent));
    img.Mutate(x => ApplyRoundedCorners(x.Fill(backgroundColor).DrawText(badge.Text, font, textColor, new PointF(badge.PaddingX, badge.PaddingY)), badge.CornerRadius));

    Response.ContentType = "image/png";

    await img.SaveAsPngAsync(Response.Body);

    return Ok();
}
```

Finally, let's see this in action!

Build up a URI that represents the badge you'd like to create and navigate to this is a browser of your choice:

```
"<BASE URI>/images/badge?text=Microsoft+Teams&textheight=10&paddingy=5&cornerradius=8&backgroundcolor=%23ffcc00&textcolor=%23333333"
```

:INFO:
All the values need to be "uri encoded" so for a css-style colour value #ffcc00 for example you'll need to encode the "#", this would become %23ffcc00 - this is also true for spaces, but you can just use "+" for that.
:END:


![Badge Image](/assets/adaptive-card-custom-ui/browser.png)

Cool, we have a dynamic badge image.

Now let's create an adaptive card that uses it. Head over to the [Adaptive Card designer](https://adaptivecards.io/designer/) and try it out.

Here is the JSON representation of the blog post card that uses this url to display tags.

:WARNING:
Remember to replace &lt;BASE URI&gt; with your own URI
:END:

```json
{
    "type": "AdaptiveCard",
    "body": [
        {
            "type": "TextBlock",
            "size": "Medium",
            "weight": "Bolder",
            "text": "New blog post"
        },
        {
            "type": "ColumnSet",
            "columns": [
                {
                    "type": "Column",
                    "items": [
                        {
                            "type": "Image",
                            "style": "Person",
                            "url": "https://avatars.githubusercontent.com/u/127280?v=4",
                            "size": "Small"
                        }
                    ],
                    "width": "auto"
                },
                {
                    "type": "Column",
                    "items": [
                        {
                            "type": "TextBlock",
                            "weight": "Bolder",
                            "text": "Scott Perham",
                            "wrap": true
                        },
                        {
                            "type": "TextBlock",
                            "spacing": "None",
                            "text": "Created 25 April, 2022",
                            "isSubtle": true,
                            "wrap": true
                        }
                    ],
                    "width": "stretch"
                }
            ]
        },
        {
            "type": "TextBlock",
            "text": "Adaptive cards custom UI",
            "wrap": true,
            "weight": "Bolder",
            "size": "Large"
        },
        {
            "type": "TextBlock",
            "text": "There is a lot you can do to make look great with what's available out of the box. There are scenarios though where you want that extra zing to make it pop, unfortunately you are only limited to the UI elements that are defined in the JSON schema, but there are some things you can do to make your cards truly unique!",
            "wrap": true
        },
        {
            "type": "ColumnSet",
            "columns": [
                {
                    "type": "Column",
                    "width": "auto",
                    "items": [
                        {
                            "type": "Image",
                            "url": "<BASE URI>/images/badge?text=Microsoft+Teams&textheight=10&paddingy=5&cornerradius=8&backgroundcolor=%23ffcc00&textcolor=%23333333"
                        }
                    ]
                },
                {
                    "type": "Column",
                    "width": "auto",
                    "items": [
                        {
                            "type": "Image",
                            "url": "<BASE URI>/images/badge?text=C%23&textheight=10&paddingy=5&cornerradius=8&backgroundcolor=%23ffcc00&textcolor=%23333333"
                        }
                    ]
                },
                {
                    "type": "Column",
                    "width": "auto",
                    "items": [
                        {
                            "type": "Image",
                            "url": "<BASE URI>/images/badge?text=Adaptive+Cards&textheight=10&paddingy=5&cornerradius=8&backgroundcolor=%23ffcc00&textcolor=%23333333"
                        }
                    ]
                }
            ]
        }
    ],
    "actions": [
        {
            "type": "Action.OpenUrl",
            "title": "Read More...",
            "url": "https://hackandchat.com"
        }
    ],
    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
    "version": "1.5"
}
```

Of course, this is all just the tip of the iceberg... now you have dynamic UI elements available to you the world is your oyster!

Happy coding