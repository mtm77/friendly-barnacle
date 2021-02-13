---
title: Generating social sharing images in Eleventy
description: Generate social sharing images for your Eleventy site without manually designing them.
date: 2021-01-10
---

This blog is built with Eleventy. When one of its posts is shared on Twitter, I wanted it to be accompanied by a social sharing image. However, I have a lot of posts, and I didn't want to design an image manually, so I decided to write some JavaScript to handle it for me. If you're looking to do the same, here's how you can set it up.

## Designing the sharing image

First, let's design the image. To do this, we'll just use HTML, CSS, and a tiny bit of JavaScript. Create a new file called _sharing-frame.liquid_. If you add this inside a subdirectory on your site, just make sure to set a permalink so it can be accessed at _localhost:8080/sharing-frame.html_.

```html
{%- raw -%}
---
layout: null
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Social sharing image generator</title>
    <style>
      * {
        padding: 0;
        margin: 0;
        box-sizing: border-box;
      }

      .background {
        width: 600px;
        height: 315px;
        color: #fff;
        background: black;
        text-align: center;
        font-weight: bold;
        font-size: 28px;
        display: flex;
        align-items: center;
      }
    </style>
  </head>
  <body>

    <div class="background">
      <div id="slot"></div>
    </div>
    
    <script>
      const queryString = window.location.search
      const urlParams = new URLSearchParams(queryString)
      const title = urlParams.get("title") || "Example"
      
      const slot = document.getElementById("slot")

      slot.innerHTML = title
    </script>

  </body>
</html>
{%- endraw -%}
```

The script at the end is important: it pulls the `title` from the URL. For instance, [visiting this URL](http://localhost:8080/sharing-frame.html?title=POST_TITLE) should show "POST_TITLE" as the title:

```
localhost:8080/sharing-frame.html?title=POST_TITLE
```

## Assembling the page data

Now that the image is ready, let's create the data to loop through to generate our images. For this example, we'll want two things: the title of the post, and the file slug for the page (which we'll use as the name for the saved image). You can add more data here to fit your use case (such as a custom image, author, date, and so on). {% note "If you want to add more data, I recommend coming back after working through this article." %}

The most straightforward way to accomplish this is to create a custom JSON file. Let's call it _sharing-pages.liquid_ and insert the following: {% note "This page can be inserted in any directory as long as it can be accessed at `localhost:8080/sharing-pages.json`." %}

```js
{%- raw -%}
---
layout: null
permalink: "sharing-pages.json"
---

{%- assign sharable_pages = collections.all | filterSharablePages -%}

[
{%- for page in sharable_pages %}
  {
    "title": "{{ page.data.title }}",
    "image": "{{ page.fileSlug | toSocialSharingImagePath }}"
  }{% unless forloop.last %},{% endunless %}
{%- endfor %}
]
{%- endraw -%}
```

In the code above, `filterSharablePages` and `toSocialSharingImagePath` are custom filters. To use custom filters in Eleventy, add them to an [.eleventy.js](https://www.11ty.dev/docs/config/) file like this: {% note "Note that this file's name begins with a period." %}

```js
module.exports = function (eleventy) {
  // Only pages with layouts can be shared. Pages without layouts,
  // such as sharing-pages.json above, won't receive images.
  eleventy.addLiquidFilter(
    "filterSharablePages",
    pages => pages.filter(page => page.data.layout)
  )

  // We'll use the path (`fileSlug`) as the name of the image.
  // The homepage ("index.html") won't have a slug, so we'll
  // just call it "index".
  eleventy.addLiquidFilter(
    "toSocialSharingImagePath",
    path => path || "index"
  )

  // ...
}
```

Now, if you visit [localhost:8080/sharing-pages.json](http://localhost:8080/sharing-pages.json), you should see your sharable pages represented as JSON.

## Programmatically screenshotting sharing images

In order to fetch our JSON data, we'll use [node-fetch](https://www.npmjs.com/package/node-fetch). And to programatically load and screenshot the page, we'll use [puppeteer](https://www.npmjs.com/package/puppeteer). These are just dev dependencies, so they won't add anything to our site in production. Add them to your project:

```bash
# If you're using npm
npm i --save-dev puppeteer node-fetch

# Or if you're using yarn
yarn add --dev puppeteer node-fetch
```

We'll use a Node script to generate our sharing images. In your site's _package.json_, add this to the scripts:

```js
scripts: {
  // ...
  "sharing-images": "node generate-sharing-images.js"
}
```

When you run `npm run sharing-images` (or `yarn run sharing-images` if you use yarn), it'll look for the file named _generate-sharing-images.js_ in the same directory and run it. Create a file with this name, and add the following:

```js
const puppeteer = require("puppeteer")
const fetch = require("node-fetch")

const PAGES_JSON = "http://localhost:8080/sharing-pages.json"
const FRAME_URL = "http://localhost:8080/sharing-frame.html?title="

async function main() {
  const response = await fetch(PAGES_JSON)
  const pages = await response.json()

  for (let i = 0; i < pages.length; i++) {
    const page = pages[i]

    const browser = await puppeteer.launch()
    const tab = await browser.newPage()
    await tab.goto(FRAME_URL + page.title)
    await tab.setViewport({ width: 600, height: 315, deviceScaleFactor: 2 })
    await tab.screenshot({ path: `images/social/${page.image}.jpg` })
    await browser.close()
    console.log(`📸 ${page.title}`)
  }
}

main()
```

## Running the script

When you want to generate your social sharing images:

1. Make sure your local Eleventy server **is running at port 8080**.

2. **In another terminal tab**, run:
    ```bash
    # If you're using npm
    npm run sharing-images

    # If you're using yarn
    yarn run sharing-images
    ```

Your images will be saved in the `/images/social/` directory as a JPG.

## Adding sharing cards to the head

You can add sharing cards in your HTML `<head>`. It could look something like this:

```html
{%- raw -%}
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:site" content="YOUR_TWITTER_USERNAME" />
<meta name="twitter:title" content="{{ title }}" />
<meta name="twitter:description" content="{{ description }}" />
<meta name="twitter:image" content="{% getSharingImage page %}" />
```

`getSharingImage` is a custom shortcode I've defined, again, inside _.eleventy.js_:

```js
// You'll want to change "example.com" out for your site's URL.
// Your index won't have a slug, so we'll hardcode it.
eleventy.addLiquidShortcode("getSharingImage", ({ fileSlug }) => (
  `https://example.com/images/social/${fileSlug || "index"}.jpg`
)
{%- endraw -%}
```

You can also add a similar implementation for [social sharing images for Facebook](https://developers.facebook.com/docs/sharing/webmasters/images/). Facebook sharing images have different size recommendations, but you can design another element on the _sharing-frame.liquid_ file and use Puppeteer to capture a second screenshot. Repeat as necessary for any images that you need.

