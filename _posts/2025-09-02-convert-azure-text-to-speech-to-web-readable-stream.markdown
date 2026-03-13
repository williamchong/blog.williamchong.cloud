---
layout: post
title: "Convert Azure text to speech API result to a web ReadableStream"
description: "Learn how to convert Azure Text-to-Speech API output into a web-compatible ReadableStream using PullAudioOutputStream for better performance and cross-platform compatibility."
date: 2025-09-02 02:00:00 +0800
lastmod: 2026-03-13
categories: code
image: /assets/images/2025-09-02-convert-azure-text-to-speech-to-web-readable-stream/cover.png
tags: javascript nodejs azure text-to-speech audio-streaming web-streams readablestream pull-stream speech-synthesis api-integration
---

![Azure text to speech API](/assets/images/2025-09-02-convert-azure-text-to-speech-to-web-readable-stream/cover.png)

## Introduction

[Previously]({% post_url 2023-10-15-convert-azure-text-to-speech-to-nodejs-stream %}), we discussed the usage of the [Azure Text-to-Speech API](https://docs.microsoft.com/en-us/azure/cognitive-services/speech-service/text-to-speech) and how to implement real-time audio streaming by converting the API's output into a stream using Node.js PassThrough. In this tutorial, we will focus on converting the API's output into a web-compatible [ReadableStream](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream).

## Why web (readable) stream?

While the Node.js stream module would work for most API use cases, using web streams provides some additional benefits:

- [Web streams](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API) are the modern standard across all JavaScript environments. This means you can use the same stream handlers in server and client code. For example, [TransformStream](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) handlers can move between API and browser code without modification, allowing for greater flexibility when design needs it.

- Compatible with the [Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API). The Fetch API natively returns a `ReadableStream`, so using web streams allows for seamless integration with the Fetch API and other web platform features.

- Unified interface across different TTS providers: By using web streams, we can create a consistent interface for different TTS providers' SDK or API, since output formats accepted in JavaScript can be converted to a `ReadableStream` one way or another.

## Pull or Push?

Azure supports [`PushAudioOutputStream`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pushaudiooutputstream) and [`PullAudioOutputStream`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pullaudiooutputstream). Both can be used to create a web-compatible `ReadableStream`. In the previous article we used `PushAudioOutputStream`, but let's revisit the technical differences between the two.

- [`PushAudioOutputStream`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pushaudiooutputstream): To create `PushAudioOutputStream`, we need to provide [`PushAudioOutputStreamCallback`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pushaudiooutputstreamcallback), which has to implement `function write(dataBuffer: ArrayBuffer)` and `function close()`.

- [`PullAudioOutputStream`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pullaudiooutputstream): To create `PullAudioOutputStream`, we need to provide [`PullAudioOutputStreamCallback`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pullaudiooutputstreamcallback), which has to implement `function read(dataBuffer: ArrayBuffer): number` and `function close()`.

For our use case, we need to mimic a `fetch` streamed response, which returns a web standard [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream). Comparing the interface of `ReadableStream`, which requires [`start`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/ReadableStream#start), [`pull`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/ReadableStream#pull), and [`cancel`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/ReadableStream#cancel) methods, we can see that `PullAudioOutputStream` is more aligned with the `ReadableStream` interface, with its `read` method corresponding to the `pull` method in `ReadableStream`.

## Implementation

For implementation, first we'll set up a [speech synthesizer](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/speechsynthesizer) similar to our [previous article]({% post_url 2023-10-15-convert-azure-text-to-speech-to-nodejs-stream %}). Note the only difference is that we will be setting up a `PullAudioOutputStream` here.

```typescript
const outputFormat =
  sdk.SpeechSynthesisOutputFormat.Audio16Khz128KBitRateMonoMp3;
const pullStream = sdk.PullAudioOutputStream.create();
const audioConfig = sdk.AudioConfig.fromStreamOutput(pullStream);
const speechConfig = sdk.SpeechConfig.fromSubscription(
  azureSubscriptionKey,
  azureServiceRegion
);
speechConfig.speechSynthesisVoiceName = voiceName;
speechConfig.speechSynthesisOutputFormat = outputFormat;

const synthesizer = new sdk.SpeechSynthesizer(speechConfig, audioConfig);
function speakTextAsync(synthesizer, text) {
  return new Promise((resolve, reject) => {
    synthesizer.speakTextAsync(
      text,
      function (result) {
        synthesizer.close();
        resolve(result);
      },
      function (err) {
        synthesizer.close();
        reject(err);
      }
    );
  });
}
await speakTextAsync(synthesizer, text);
```

Then we'll create a `ReadableStream` from the `PullAudioOutputStream`. We only need to implement `pull` and `cancel`. Since the read method requires an [`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer), we'll create a new buffer for each read operation.

```typescript
const readableStream = new ReadableStream({
  async pull(controller) {
    try {
      const buffer = new ArrayBuffer(1024);
      const bytesRead = await pullStream.read(buffer);

      if (bytesRead > 0) {
        const chunk = new Uint8Array(buffer, 0, bytesRead);
        controller.enqueue(chunk);
      } else {
        pullStream.close();
        controller.close();
      }
    } catch (error) {
      console.error("[Speech] Error reading from pull stream:", error);
      controller.error(error);
      pullStream.close();
    }
  },
  cancel() {
    pullStream.close();
  },
});

return readableStream;
```

One minor optimization we can make is to use the [`type`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/ReadableStream#type) and [`autoAllocateChunkSize`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream/ReadableStream#autoallocatechunksize) options on the `ReadableStream`. This allows the stream to reuse buffers from existing allocations or allocate new ones as needed to minimize memory usage and improve performance.

```typescript
const readableStream = new ReadableStream({
  type: "bytes",
  autoAllocateChunkSize: 1024,
  async pull(controller) {
    try {
      const byobRequest = controller.byobRequest;
      if (byobRequest?.view) {
        const buffer = byobRequest.view.buffer as ArrayBuffer;
        const bytesRead = await pullStream.read(buffer);

        if (bytesRead > 0) {
          byobRequest.respond(bytesRead);
        } else {
          pullStream.close();
          controller.close();
        }
      } else {
        // Fallback: allocate our own buffer
        const buffer = new ArrayBuffer(1024);
        const bytesRead = await pullStream.read(buffer);

        if (bytesRead > 0) {
          const chunk = new Uint8Array(buffer, 0, bytesRead);
          controller.enqueue(chunk);
        } else {
          pullStream.close();
          controller.close();
        }
      }
    } catch (error) {
      console.error("[Speech] Error reading from pull stream:", error);
      controller.error(error);
      pullStream.close();
    }
  },
  cancel() {
    pullStream.close();
  },
});

return readableStream;
```

Then we can proceed to read from the `readableStream` as needed.

## Conclusion

In this blog post, we explored how to convert Azure Text-to-Speech output into a web-readable stream. By leveraging the [`PullAudioOutputStream`](https://docs.microsoft.com/en-us/javascript/api/microsoft-cognitiveservices-speech-sdk/pullaudiooutputstream) and the [ReadableStream API](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream), we can create a flexible and efficient solution for handling audio data in web applications. This approach not only improves performance but also enhances maintainability by providing a clear separation of concerns between the TTS provider's interface and the API response format.
