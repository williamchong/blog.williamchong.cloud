---
layout: post
title: "Handling Minimax TTS API: Basic HTTP and Streaming"
description: "Complete guide to integrating Minimax Text-to-Speech API with both blocking and streaming approaches. Learn how to implement basic TTS integration and convert it to real-time streaming for better user experience."
date: 2025-06-22 02:00:00 +0800
categories: code
image: /assets/images/2025-06-22-handling-minimax-tts-api-basic-and-streaming/cover.png
tags: minimax text-to-speech tts api-integration server-sent-events sse streaming real-time audio nodejs javascript
---

![Minimax TTS API Integration](/assets/images/2025-06-22-handling-minimax-tts-api-basic-and-streaming/cover.png)

> **Update**: This post has a follow-up — [Minimax TTS API Update: let's vibe a TypeScript SDK]({% post_url 2026-02-13-minimax-tts-api-improvements-eventsource-parser-and-sdk %}) — covering `eventsource-parser` for spec-compliant SSE parsing, the new `exclude_aggregated_audio` option, and the [`minimax-speech-ts`](https://github.com/williamchong/minimax-speech-ts) TypeScript SDK.

## Introduction

Continuing from our series of integrating TTS (Text-to-Speech) API services from [Google](/2023/10/13/convert-google-text-to-speech-to-nodejs-stream/), [Azure](/2023/10/15/convert-azure-text-to-speech-to-nodejs-stream/) and [AWS](/2025/06/06/convert-aws-polly-to-nodejs-stream/), the latest attempt in my TTS exploration is [Minimax](https://minimax.io). Minimax is a relatively new company and unlike previous attempts, it is not a full blown cloud service provider, but rather a specialized API service for AI, video and audio related tasks.

Minimax provides three ways to interact with their [TTS API](https://www.minimax.io/platform/document/T2A%20V2?key=66719005a427f0c8a5701643#TJeyxusWAUP0l3tX67brbAyE): WebSocket, HTTP and MCP. Since I'm working on a server application, I'll focus on the HTTP API. For the HTTP API, Minimax offers 2 modes: streaming and non-streaming. In this post I'll cover both approaches, with the goal of implementing a complete streaming API with low latency and high performance.

## Integrating Minimax TTS API

Unfortunately, Minimax doesn't provide an official SDK, so we'll use the HTTP API directly. The API is well documented, and we can use any HTTP client to interact with it. In this example, I'll use the `$fetch` function from Nuxt 3, but you can use any HTTP client of your choice.

### The Non-streaming API

To begin, we'll implement the basic TTS API integration using the non-streaming API. This means we wait for the audio to be completely generated, then download the entire audio as a buffer. The implementation is simple and straightforward:

```typescript
const response = await $fetch(
  `https://api.minimaxi.chat/v1/t2a_v2?GroupId=${minimaxGroupId}`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${minimaxAPIKey}`,
    },
    body: {
      text: "your text",
      model: "speech-02-hd",
      voice_setting: {
        voice_id: "Chinese (Mandarin)_Warm_Bestie",
        speed: 0.95,
        pitch: -1,
        emotion: "neutral",
      },
      language_boost: "Chinese,Yue",
    },
  }
);
const audioHex = response.data.audio;
const audioBuffer = Buffer.from(audioHex, "hex");
```

Here we're calling the [`speech-02-hd` model](https://www.minimax.io/news/minimax-speech-02) with a preference for `Chinese,Yue`. This is required for my case because Mandarin Chinese and Cantonese text often look very similar and the target spoken language cannot be reliably determined just by text. High quality Cantonese voice support is one of the unique strength of Minimax. Note that even if the voice is set to `Chinese (Mandarin)_Warm_Bestie`, the audio can still be generated in Cantonese, and any English inside the text will still be pronounced as English. This behaviour is similar to the Azure multilingual voices and is very convenient when handling multilingual texts.

### Streaming the Non-streaming API

Even if the non-streaming API is not designed for streaming, we can still convert the complete received response into a stream. This allow user to start playing the audio as soon as our API start sending the response, rather than waiting for the entire audio to be downloaded in browser.

```javascript
// continue from the previous code snippet
const audioBuffer = Buffer.from(audioHex, "hex");
const stream = Readable.from(audioBuffer);
return sendStream(event, stream);
```

Another optimization we could do is to receive the TTS API response as a stream, and then pipe it to the HTTP response. However we would be still bounded by the fact that the TTS API would not start responding until the entire audio is generated. A much better approach is to use the streamed version of the API.

### Implement Real streaming using the streamed API

According to the [Minimax documentation](https://www.minimax.io/platform/document/T2A%20V2?key=66719005a427f0c8a5701643#TJeyxusWAUP0l3tX67brbAyE), the streamed API has the following response format:

```
//end
{
    "data":{
        "audio":"hex audio_chunk1 + hex audio_chunk2 + hex audio_chunk3",
        "status":2
    },
     "extra_info":{
      ...
    },
    "trace_id":"01b8bf9bb7433cc75c18eee6cfa8fe21",
    "base_resp":{
        "status_code":0,
        "status_msg":""
    }
}
// thrid chunk
{
    "data":{
        "audio":"hex audio_chunk3",
        "status":1,
    },
    "trace_id":"01b8bf9bb7433cc75c18eee6cfa8fe21",
    "base_resp":{
        "status_code":0,
        "status_msg":""
    }
}
//second chunk
{
    "data":{
        "audio":"hex audio_chunk2",
        "status":1,
    },
    "trace_id":"01b8bf9bb7433cc75c18eee6cfa8fe21",
    "base_resp":{
        "status_code":0,
        "status_msg":""
    }
}
//first chunk
{
    "data":{
        "audio":"hex audio_chunk1",
        "status":1,
    },
    "trace_id":"01b8bf9bb7433cc75c18eee6cfa8fe21",
    "base_resp":{
        "status_code":0,
        "status_msg":""
    }
}
```

At first glance, this response format is not trivial to parse. Due to the nature of network streaming, we cannot assume that a response will always be received in a single JSON chunk. This means we need to keep reading the stream until we receive a complete JSON object. Since we might receive multiple JSON objects in a single response, the parentheses matching of `{` and `}` must be checked when storing the response in a buffer. We also need to trim away any parts from the next JSON object that might be included in the current buffer to ensure `JSON.parse()` doesn't throw an error.

The last chunk of the response is a summary block, which can be ignored for audio streaming purposes. It would be nice if the API allowed us to opt out of receiving the summary block, but since that's not possible, we'll simply drop it.

```
data: {"data":{"audio":"(hex)...... \n\n
data: ......(hex)","status":1,},"trace_id":"01b8bf9bb7433cc75c18eee6cfa8fe21","base_resp":{"status_code":0,"status_msg":""}}\n\n
```

Fortunately, by investigating the actual formatting and behavior of the streamed API response, the server always chunks its response starting with `data: ` and ending with `\n\n`. This clearly indicates that the streamed API response is implemented as Server-Sent Events (SSE). This means we can make assumptions about parsing the streamed response, or even use a library that handles SSE for us. In this post, I'll handle the response parsing manually, checking for `data: ` and `\n` to extract the audio chunks.

Let's implement the streaming API in our Nuxt 3 server:

```typescript
const response = await $fetch<ReadableStream>(
  `https://api.minimaxi.chat/v1/t2a_v2?GroupId=${minimaxGroupId}`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${minimaxAPIKey}`,
    },
    responseType: "stream", // set this to receive a stream in `response`
    body: {
      stream: true, // ask for streaming response
      text: "your text",
      model: "speech-02-hd",
      voice_setting: {
        voice_id: "Chinese (Mandarin)_Warm_Bestie",
        speed: 0.95,
        pitch: -1,
        emotion: "neutral",
      },
      language_boost: "Chinese,Yue",
    },
  }
);
```

To convert the received hex into a stream, we need to read the stream and parse the response as it comes in. We'll use [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) since fetch returns a [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) when `responseType` is set to `stream`. [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) allows us to convert the incoming stream into a format suitable for sending to the client in a streamed format.

First, we need to convert the [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) (which contains uint8 array chunks) into a string format that we can parse. This is straightforward with the [`TextDecoderStream`](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoderStream) API.

```typescript
const decodedStream = response.pipeThrough(new TextDecoderStream()); // Now the stream is in string format
```

Next, we'll create the actual [`TransformStream`](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream) logic. We need to implement `start`, `transform`, and `flush` methods. The main logic will be in the `transform` method. Since we know a complete SSE event is separated by `\n\n`, we can use this to determine if we need to fetch more data or if we already have a complete event to process. Note that this assumes no `\n\n` is present in the audio data itself, which is a safe assumption for audio data in hex format.

```typescript
const processStream = new TransformStream({
  start() {
    buffer = "";
  },
  transform(chunk, controller) {
    try {
      buffer += chunk;
      let eventEndIndex = buffer.indexOf("\n\n");
      while (eventEndIndex !== -1) {
        const event = buffer.substring(0, eventEndIndex).trim();
        buffer = buffer.substring(eventEndIndex + 2); // '\n\n' is 2 characters long
        if (event) {
          try {
            const audioBuffer = processEventData(event);
            if (audioBuffer) {
              controller.enqueue(audioBuffer);
            }
          } catch (error) {
            controller.error(
              (error as Error).message || "TTS_PROCESSING_ERROR"
            );
            return;
          }
        }
        eventEndIndex = buffer.indexOf("\n\n");
      }
    } catch (error) {
      controller.error(
        "TTS_PROCESSING_ERROR: Failed to process text-to-speech data"
      );
      return;
    }
  },
  flush(controller) {
    if (!buffer.trim()) {
      return;
    }
    try {
      const audioBuffer = processEventData(buffer);
      if (audioBuffer) {
        controller.enqueue(audioBuffer);
      }
    } catch (error) {
      // no op
    }
  },
});
```

In `processEventData`, we parse the actual event data, removing the SSE `data: ` prefix and `\n\n` suffix. We extract the hex audio data into a `Buffer` and return it. The buffer is then sent to the client using `controller.enqueue()`.

We also check for `data.status === 1` to ensure the summary block at the end of the stream is ignored.

```typescript
function processEventData(eventData: string) {
  const dataMatch = eventData.match(/^data:\s*(.+)$/m);
  if (!dataMatch) return null;

  const jsonStr = dataMatch[1].trim();
  if (!jsonStr) return null;

  const parsed: TTSChunk = JSON.parse(jsonStr); // this might throw if the JSON is malformed

  if (parsed.base_resp?.status_code !== 0) {
    throw createError({
      name: "TTS_API_ERROR",
      message: parsed.base_resp?.status_msg || "Unknown API error",
    });
  }

  if (parsed.data.status === 1 && parsed.data.audio) {
    return Buffer.from(parsed.data.audio, "hex");
  }

  return null;
}
```

Finally, we can pipe the transform stream to `sendStream` and send a binary audio stream to the client:

```typescript
// continue from the previous code snippet
// decodedStream is a ReadableStream of strings
// processStream is a TransformStream that processes the SSE events
decodedStream.pipeThrough(processStream);
return sendStream(event, processStream.readable);
```

This approach involves some native assumptions about event formatting, but it works reasonably well as an integration example and MVP for this task.

## Conclusion

In this post, we explored how to integrate the Minimax TTS API using both non-streaming and streaming approaches. The non-streaming method provides a straightforward way to get audio data, suitable for short texts or applications where latency isn't a concern. However, for applications requiring real-time audio playback, the streaming approach, although more complex, offers a significant improvement in user experience by allowing audio to be played as it's generated.

The streaming implementation demonstrates how to handle Server-Sent Events manually and convert hex audio chunks into a streamable format. This approach provides low latency and improved perceived performance, making it ideal for interactive applications where immediate audio feedback is important.

## Additional Resources

### Related TTS Integration Articles

- [Convert Google text to speech API result to HTTP streamed response](/2023/10/13/convert-google-text-to-speech-to-nodejs-stream/)
- [Convert Azure text to speech API result to HTTP streamed response](/2023/10/15/convert-azure-text-to-speech-to-nodejs-stream/)
- [Convert AWS Polly text to speech API result to HTTP streamed response](/2025/06/06/convert-aws-polly-to-nodejs-stream/)
- [Convert OpenAI API stream to HTTP streamed response](/2023/10/28/openai-api-to-http-streamed-response/) (General streaming concepts)

### Documentation

- [Minimax TTS API Documentation](https://www.minimax.io/platform/document/T2A%20V2?key=66719005a427f0c8a5701643#TJeyxusWAUP0l3tX67brbAyE)
- [Server-Sent Events - MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [ReadableStream - MDN](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream)
- [TransformStream - MDN](https://developer.mozilla.org/en-US/docs/Web/API/TransformStream)
- [TextDecoderStream - MDN](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoderStream)
