---
layout: post
title: "Integrating Vertex AI Search for Commerce Part 2: Collecting Realtime Data"
description: "Learn how to implement real-time user event collection for Google Vertex AI Search for Commerce, with practical solutions to the challenges of the JavaScript Pixel tracking method and a ready-to-use implementation for Nuxt applications."
date: 2025-02-02 04:00:00 +0800
lastmod: 2026-03-13
categories: code
image: /assets/images/2025-02-02-vertex-ai-search-retail-2-collecting-real-time-events/0.png
tags: google-cloud vertex-ai retail-search commerce javascript-pixel tracking-implementation nuxt event-collection product-recommendation real-time-data e-commerce
---

![Google Cloud Vertex AI Search for Commerce](/assets/images/2025-02-02-vertex-ai-search-retail-2-collecting-real-time-events/0.png)

## Background

In the [previous post]({% post_url 2024-11-16-vertex-ai-search-retail-1-importing-data %}), we covered the initial steps to set up [Vertex AI Search for Commerce](https://cloud.google.com/solutions/vertex-ai-search-commerce) (yes, they have changed their product name again, from Retail to Commerce). For some use cases, importing an existing product catalog and user data may be sufficient to build the data model.

However, if you lack enough existing data to train the model or need fresh data to confirm that your model is functioning as expected, you must feed user events to the Vertex AI Search service as they occur on your website. In this post, we will discuss how to integrate real-time event collection into your site.

## Collecting Real-time Events

Google provides a [documentation page](https://cloud.google.com/retail/docs/record-events) specifically on how to record user events. Generally, there are three methods: Google Tag Manager (GTM), API, and JavaScript Pixel. You can find more details here.

A list of recommended user events and their payloads can be found [here](https://cloud.google.com/retail/docs/user-events#formats).

If you are already using GA4 events, some of these can automatically convert to retail user events, as detailed [here](https://cloud.google.com/retail/docs/user-events#ga4-mapping).

However, some retail user event types do not have a direct equivalent in GA4, which means you will need to either ignore those event types or send them manually through another method.

### The Easy Way - GTM & GA4

If you already have GTM set up on your website, this should be the simplest method. The GTM console includes templates like `Variable - Ecommerce` and `Variable - Cloud Retail`. Refer to the GTM documentation for straightforward implementation.

Unfortunately, we do not have GTM set up on our website due to complications with running it alongside Content Security Policy (CSP), so I will skip this part.

### API - For Those with GA4 or Concerned About Ad Blockers

If you are familiar with implementing server-side events (like in [Meta's Conversions API](https://developers.facebook.com/docs/marketing-api/conversions-api/)), you can manually build your JSON payload and use the [`userEvents.write` API to send events](https://cloud.google.com/retail/docs/record-events#write) to the Vertex AI Search service. This ensures that events are sent even if the user has ad blockers or has disabled JavaScript, provided you have the appropriate server-side API related to the event.

Additionally, if you are already using Google Analytics 4 (GA4), you can use a `prebuilt_rule` called `ga4_bq` to send a GA4 event payload directly to the `userEvents.write` API. While this is mentioned in the documentation, I'm unsure in what scenarios this would be applicable since you cannot directly retrieve a GA4 event payload from `gtm.js`. Please let me know if you have ideas!

### JavaScript Pixel Tracking

For me, the most suitable method appears to be using the JavaScript Pixel. Similar to GA4 or other pixel tracking services, I would simply include a script and call functions when a user event occurs.

The documentation provides a complete JavaScript example:

```javascript
var user_event = {
  eventType: "detail-page-view",
  visitorId: "visitor-id",
  userInfo: {
    userId: "user-id",
  },
  attributionToken: "attribution-token",
  experimentIds: "experiment-id",
  productDetails: [
    {
      product: { id: "123" },
    },
  ],
};

var _gre = _gre || [];
// Credentials for project.
_gre.push(["apiKey", "api-key"]);
_gre.push(["logEvent", user_event]);
_gre.push(["projectId", "project-id"]);
_gre.push(["locationId", "global"]);
_gre.push(["catalogId", "default_catalog"]);

(function () {
  var gre = document.createElement("script");
  gre.type = "text/javascript";
  gre.async = true;
  gre.src = "https://www.gstatic.com/retail/v2_event.js";
  var s = document.getElementsByTagName("script")[0];
  s.parentNode.insertBefore(gre, s);
})();
```

While this seems straightforward, I have some bad feeling looking at it. The example covers the code for configuration, event formatting, and script loading, but why are the script tags created after all the event formatting and function calls? What function from `window._gre` should I invoke after the script has loaded?

## Retail Pixel - It Doesn't Work as Expected

From the example code above, it seems that the retail pixel behaves like GTM or any other pixel tracking service. My expectations were:

A global variable under window is created as an array, in this case, `_gre`.
The array is used to store configuration and function calls before the script loads.
The actual script loads asynchronously. After the script loads, it should read the `_gre` array and execute any configuration or function calls stored within.
Finally, the `_gre` should be replaced by a new object with the same name, implementing functions that allow interaction with the retail API.
However, parts of this process are not shown in the example code, particularly what to do after the script is loaded. One might expect to either call a new function like `_gre.logEvent`, or continue using `_gre.push` to send user events via the pixel. Unfortunately, such functionality does not exist, and the example code appears to encompass the entire functionality of the retail pixel.

### Configuration Values Are Not Stored in Pixel

Typically, one would expect configuration values like:

```javascript
_gre.push(['apiKey', 'api-key']);
_gre.push(['projectId', 'project-id']);
_gre.push(['locationId', 'global']);
_gre.push(['catalogId', 'default_catalog']);
```

to be stored in the pixel object. Any subsequent calls to `logEvent` (or its `_gre.push` equivalent) would then be able to read these values. However, these values are not retained in the pixel object, meaning they must be provided each time a user event is sent.

### `_gre.push` Is Not Hooked by the Script

User events are sent by invoking `_gre.push(['logEvent', user_event])`. This works if `_gre.push` is called before the pixel JavaScript is loaded. However, if it is called after the script has loaded, it will not function, as the `_gre.push` function is not hooked after the script is loaded. In GTM, pushing something into the data layer should trigger GTM events because it hooks into the `.push` function of data layer variables. Unfortunately, this is not the case for the retail pixel; you must manually trigger another function to send the user event after the script is loaded.

### `_gre.logEvent` Is Not a Function

Continuing from the previous point, if `.push` does not work, what would the trigger function be? It appears that `logEvent` is a function name that should be called. However, it is not a function, and calling` _gre.logEvent` results in an error.

## Simple Analysis of the Retail Pixel

To resolve these mysteries, one must examine [the source code of the retail pixel](https://www.gstatic.com/retail/v2_event.js).

It turns out the function is indeed called `logEvent`, but it is encapsulated within a new variable called `cloud_retail`. This `cloud_retail.logEvent` function takes the entire `_gre` as a parameter, and is only invoked once after the script has loaded!

To send a user event after the script is loaded, you must call `_gre.push(['logEvent', user_event])` and then call `cloud_retail.logEvent(_gre)` after the script is loaded.

However, note that the `_gre` array is not cleared after the script loads, so if there are existing events in `_gre` before the script loads, they will also be sent if you call `logEvent(_gre)` again, resulting in duplicate events. For `cloud_retail.logEvent(_gre)` to function correctly, you must clear `_gre` after invoking it to prevent this.

Another issue arises: `_gre` or the pixel does not store configuration values like project ID or catalog ID, meaning these values must be passed again to `_gre`.

Thus, to log events using `_gre` and `cloud_retail.logEvent` naively, the code would look like this:

```javascript
_gre = [];
_gre.push(["apiKey", "api-key"]);
_gre.push(["projectId", "project-id"]);
_gre.push(["locationId", "global"]);
_gre.push(["catalogId", "default_catalog"]);
_gre.push(["logEvent", user_event]);
cloud_retail.logEvent(_gre);
```

## Hack, or "actual implementation"

A more practical solution would be to just forget `_gre`, store these config values in a separate object, and pass it directly to `cloud_retail.logEvent` every time an event is sent.

```javascript
function logEvent(user_event) {
  // get api-key, project-id, global and default_catalo from some where
  cloud_retail.logEvent([
    ["apiKey", "api-key"],
    ["projectId", "project-id"],
    ["locationId", "global"],
    ["catalogId", "default_catalog"],
    ["logEvent", user_event],
  ]);
}
```

Consequently, I developed this class for my Nuxt project:

```javascript
class GoogleRetailPixel {
  userId;
  visitorId;
  apiKey;
  projectId;

  constructor(apiKey, projectId) {
    this.apiKey = apiKey;
    this.projectId = projectId;
    window._gre = window._gre || [];
    window._gre.push(["apiKey", apiKey]);
    window._gre.push(["projectId", projectId]);
    window._gre.push(["locationId", "global"]);
    window._gre.push(["catalogId", "default_catalog"]);
  }

  setUserId(userId) {
    this.userId = userId;
  }

  setVisitorId(visitorId) {
    this.visitorId = visitorId;
  }

  logEvent(eventType, payload = {}, { attributionToken, experimentIds } = {}) {
    if (!this.visitorId) {
      return;
    }
    const event = {
      eventType,
      attributionToken,
      experimentIds,
      visitorId: this.visitorId,
      userInfo: this.userId
        ? {
            userId: this.userId,
          }
        : undefined,
      ...payload,
    };
    // HACK: cloud_retail does not replace _gre on init
    // cloud_retail only calls logEvent once on _gre and
    // it does not even clear _gre after that
    if (window.cloud_retail) {
      window.cloud_retail.logEvent([
        ["apiKey", this.apiKey],
        ["projectId", this.projectId],
        ["locationId", "global"],
        ["catalogId", "default_catalog"],
        ["logEvent", event],
      ]);
    } else {
      window._gre.push(["logEvent", event]);
    }
  }
}
```

With this implementation, I push events into `_gre` if the `cloud_retail` object is not yet loaded, and call `cloud_retail.logEvent` if it is. This prevents duplicate events from being sent while allowing the asynchronous loading of the retail pixel.

I have also wrapped this functionality [as a Nuxt 2 library](https://www.npmjs.com/package/nuxt-gre-pixel-module), just in case I need to use it in other projects.

Since the project requiring this is still running Nuxt 2, I haven't written a Nuxt 3 version yet, but it should be straightforward and may happen if we migrate to Nuxt 3.

## Confirming Event Transmission

![User Events Integration page](/assets/images/2025-02-02-vertex-ai-search-retail-2-collecting-real-time-events/0.png)
To confirm that the events are being sent, you can check the User Events Integration page in the Vertex AI Search console, specifically within the Event Tab on the Data Page. This section displays real-time event counts from the specified date to the present. You can also monitor the percentage of unjoined events to ensure you are sending correct product IDs or to determine if the product catalog needs updating or importing.

## Conclusion

It seems that using the retail pixel is not as straightforward as I initially thought. The pixel has limited functionality and does not appear as production-ready as other Google products or similar services named "pixel". However, with a few tweaks and hacks, it can still function as expected.

![Data Error when training model](/assets/images/2025-02-02-vertex-ai-search-retail-2-collecting-real-time-events/1.png)

Now that we have real-time events being sent to the Vertex AI Search service, I hope that my model requirements will be met, allowing me to start training my model. However, even if I meet the data requirements stated on the model training page, the training could still fail midway due to insufficient data.

As this issue is challenging for me to debug as a Google Cloud user, I will likely wait to see if more data collected over time resolves the problem or if I can find the time to contact someone from Google Cloud to investigate. Stay tuned for any new posts in case I have further findings.
