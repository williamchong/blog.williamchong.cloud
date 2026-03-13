---
layout: post
title: "Fix Google Analytics (GA4) Events Not Firing in iFrame"
description: "A comprehensive troubleshooting guide explaining why Google Analytics (GA4) events fail to trigger in cross-origin iframes, with a proven solution using proper SameSite cookie settings to ensure accurate event tracking."
date: 2025-02-19 23:00:00 +0800
categories: code
image: /assets/images/2025-02-19-fix-ga-events-in-iframe/0.png
tags: google-analytics ga4 cookies samesite cross-origin iframe web-tracking debugging browser-security chrome third-party-cookies web-development
---

![Debugging GA4 Events](/assets/images/2025-02-19-fix-ga-events-in-iframe/0.png)

## Background

Although I’m not a Google Cloud Platform or Google Analytics expert, I often encounter strange issues that neither ChatGPT nor StackOverflow can resolve. This is one of those cases.

We were working on an HTML widget designed to be embedded in partner websites as an `<iframe>`. Google Analytics 4 (GA4) tracking was added to the widget, but we noticed that the number of logged page view events was significantly lower than expected.

Initially, we suspected that ad-blockers were causing the problem. However, after comparing the GA4 data from the partner's site, we ruled this out. The page views on the parent site were normal and much higher than those logged from the iframe.

It is worth mentioning that this issue is not about triggering iframe GA4 events from a cross-origin parent site, which can be resolved using [postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage).

---

## Troubleshooting the Issue in Production

### Initial Checks

To eliminate implementation issues, we first confirmed that GA4 events were being sent correctly when the widget was accessed directly (not embedded). The simplest way to do this was to open the widget's `src` URL directly, and check the network tab in the browser's developer console. By filtering network requests using the GA4's tag ID, we could easily inspect outgoing events from `gtag`.

![Network requests in page](/assets/images/2025-02-19-fix-ga-events-in-iframe/1.png)

Here, we see two requests: one for loading the `gtag` JavaScript and another for sending the `page_view` event. This confirms that GA4 events are triggered correctly when the widget is opened directly.

Next, we checked what happens when the widget was embedded in an `iframe`. By inspecting the `iframe` element on the parent site, we observed the following:

![Network requests in iframe](/assets/images/2025-02-19-fix-ga-events-in-iframe/2.png)

The `gtag.js` script was loaded correctly, but the `page_view` event was not being sent!

---

### Manually Triggering GA4 Events from the iFrame

To determine whether the issue was related to JavaScript, we attempted to manually trigger a `page_view` event. By default, `gtag` automatically sends a `page_view` event upon initialization. To test if the automated event was being blocked, we manually triggered it with the following code:

```javascript
console.log(gtag); // Sanity check to ensure gtag is loaded
gtag("event", "page_view");
```

First, we tested this in the non-embedded version of the widget:

![Console in page](/assets/images/2025-02-19-fix-ga-events-in-iframe/3.png)

As expected, after a few seconds, a new network call is triggered, indicating that the `page_view` event was sent successfully. The few seconds delay is due to the default [GA4 event batching](https://support.google.com/analytics/answer/9322688?hl=en#zippy=%2Crealtime-report%2Cdebugview-report:~:text=Understand%20event%20grouping).

Let's try it in the embedded version. Note that we can select the iframe in the console to run the code within the `iframe` context.

![Console in iframe](/assets/images/2025-02-19-fix-ga-events-in-iframe/4.png)
Nothing happened! The event was not sent, indicating that something was blocking it.

### Using the GA Debugger

To dig deeper, we used the [Google Analytics Debugger Chrome extension](https://chromewebstore.google.com/detail/google-analytics-debugger/jnkmfdileelhofjcijamephohjechhna), which logs all GA events to the console. After installing and enabling the extension, we refreshed the page.

For sanity checks, we ran it on the non-embedded version first:

![GA Debugger console](/assets/images/2025-02-19-fix-ga-events-in-iframe/5.png)

The event was triggered successfully, and its details were logged in the console.

Next, we tested the embedded version:

![GA Debugger console in iframe](/assets/images/2025-02-19-fix-ga-events-in-iframe/6.png)

The event details were missing, and we saw two warnings in the console:

- Unable to update session cookie.
- Unable to set cookie.

It became clear that these cookie-related warnings were preventing the event from being sent.

## Cookies in Cross-Origin iFrames and SameSite

The root cause of the problem was [Chromium's changes](https://chromestatus.com/feature/5088147346030592) to the `SameSite` cookie attribute. [SameSite](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#samesitesamesite-value) controls whether cookies should be sent in cross-site requests. In this case, since the iframe's domain differed from the parent site's domain, cookies were blocked by default.

To mitigate this issue, the `SameSite=None` attribute must be used to allow cross-origin cookies.

## Fixing the Cookie Issue

To ensure proper cookie settings for `gtag` and GA4, the following configuration flags should be added during `gtag` initialization:

```javascript
gtag("config", "GA_MEASUREMENT_ID", {
  cookie_flags: "SameSite=None; Secure",
  cookie_update: false,
});
```

Here’s the result after implementing these flags:

![GA Debugger in iframe after fix](/assets/images/2025-02-19-fix-ga-events-in-iframe/7.png)

With this fix, the `page_view` events were successfully sent from the iframe.

## Conclusion

While I knew of the `SameSite` cookie changes, I didn’t expect them to affect simple GA4 `page_view` events from cross-origin iframes that don’t interact with their parent site. It would be helpful if GA4 provided better fallback mechanisms for events when cookies are blocked—or at the very least, documented this behavior more clearly.

This raises an interesting question: are advertising agencies that embed ads in iframes aware of this issue? I’m not sure. Hopefully, this post won’t degrade everyone’s privacy by revealing this workaround, but it might help others struggling with similar problems.
