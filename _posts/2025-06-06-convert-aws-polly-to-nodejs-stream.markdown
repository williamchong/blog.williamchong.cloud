---
layout: post
title: "Convert AWS Polly text to speech API result to HTTP streamed response"
description: "A simple guide on implementing HTTP streaming of AWS Polly text-to-speech output for improved user experience and reduced latency."
date: 2025-06-06 10:00:00 +0800
categories: code
image: /assets/images/2025-06-06-convert-aws-polly-to-nodejs-stream/cover.jpg
tags: javascript nodejs aws polly text-to-speech audio-streaming speech-synthesis
---

![AWS Polly text to speech API](/assets/images/2025-06-06-convert-aws-polly-to-nodejs-stream/cover.jpg)

In previous articles, I've explored streaming implementations for [Google's Text-to-Speech API]({% post_url 2023-10-13-convert-google-text-to-speech-to-nodejs-stream %}) and [Azure's Text-to-Speech service]({% post_url 2023-10-15-convert-azure-text-to-speech-to-nodejs-stream %}). Continuing this series, let's see how to implement streaming with AWS Polly, Amazon's text-to-speech service.

Spoiler: It's trivial.

## AWS Polly Javascript SDK

The command we would be using is [SynthesizeSpeechCommand](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/polly/command/SynthesizeSpeechCommand/) from the AWS SDK for JavaScript v3. The response from the `SynthesizeSpeechCommand` already includes an `AudioStream`. `AudioStream` has methods `transformToByteArray`, `transformToString`, and `transformToWebStream`, which transform the audio stream into different formats. For HTTP streaming, we can just use `transformToWebStream()` to convert the AWS SDK audio stream into a web-compatible stream.

## Example code

The following code converts a text input to an ogg audio stream using AWS Polly's `SynthesizeSpeechCommand`.

```javascript
import type {
  LanguageCode,
  VoiceId,
} from '@aws-sdk/client-polly'
import {
  PollyClient,
  SynthesizeSpeechCommand,
} from '@aws-sdk/client-polly'

const client = new PollyClient({
  region: awsRegion,
  credentials: {
    accessKeyId: awsAccessKeyId,
    secretAccessKey: awsAccessKeySecret,
  },
})

...

const command = new SynthesizeSpeechCommand({
  Text: text,
  OutputFormat: 'ogg_vorbis',
  VoiceId: 'Ruth' as VoiceId,
  LanguageCode: 'en-US' as LanguageCode,
  Engine: 'neural',
  TextType: 'text',
})
const response = await client.send(command)
if (!response.AudioStream) {
  throw createError({
    status: 500,
    message: 'SPEECH_SYNTHESIS_FAILED',
  })
}
const stream = response.AudioStream.transformToWebStream()
setHeader(event, 'content-type', 'audio/ogg; codecs=opus')
setHeader(event, 'cache-control', 'public, max-age=3600')
return sendStream(event, stream)

```

## Conclusion

Since AWS SDK already provides a stream based response and useful helper methods to convert the audio stream, implementing HTTP streaming for AWS Polly is straightforward. This allows you to reduce the delay between the request and the audio playback, enhancing the overall user experience with real-time audio playback.
