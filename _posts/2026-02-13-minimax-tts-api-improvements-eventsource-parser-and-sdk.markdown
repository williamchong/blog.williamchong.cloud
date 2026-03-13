---
layout: post
title: "Minimax TTS API Update: let's vibe a TypeScript SDK"
description: "Follow-up to Minimax TTS streaming integration. How I vibe-coded minimax-speech-ts from API docs to npm package in a day using Claude Code, and the Context-Limit-Progress framework for AI-assisted coding."
date: 2026-02-13 02:00:00 +0800
lastmod: 2026-02-13
categories: code
image: /assets/images/2026-02-13-minimax-tts-api-improvements-eventsource-parser-and-sdk/cover.png
tags: minimax text-to-speech tts api-integration server-sent-events sse streaming nodejs typescript sdk eventsource-parser claude-code ai-coding vibe-coding
---

![Minimax TTS API Improvements](/assets/images/2026-02-13-minimax-tts-api-improvements-eventsource-parser-and-sdk/cover.png)

## Introduction

In the [previous post]({% post_url 2025-06-22-handling-minimax-tts-api-basic-and-streaming %}), we implemented streaming text-to-speech with the Minimax API by manually parsing Server-Sent Events and handling the aggregated summary block at the end of the stream. While functional, that approach had two pain points: hand-rolled SSE parsing logic and the need to detect and discard the final summary chunk. Since then, three things have improved the situation significantly:

1. Minimax added `stream_options.exclude_aggregated_audio` to the API
2. The [`eventsource-parser`](https://github.com/rexxars/eventsource-parser) library provides a robust, spec-compliant SSE parser
3. Most importantly, I published [`minimax-speech-ts`](https://github.com/williamchong/minimax-speech-ts), a TypeScript SDK that wraps the entire Minimax Speech API (Lol)

## Excluding the Aggregated Audio

In the previous post, we had to handle a summary block at the end of the stream that contained the complete aggregated audio data. We checked for `data.status === 1` to filter out the final chunk with `status === 2`. This was necessary because the API always sent the full concatenated audio as the last event, which we didn't need when streaming.

Minimax has since added a [`stream_options.exclude_aggregated_audio`](https://platform.minimax.io/docs/api-reference/speech-t2a-http#body-stream-options-exclude-aggregated-audio) parameter. When set to `true`, the final chunk no longer contains the complete audio data. This means we no longer need to filter it out ourselves:

```typescript
const response = await $fetch<ReadableStream>(
  `https://api.minimaxi.chat/v1/t2a_v2?GroupId=${minimaxGroupId}`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${minimaxAPIKey}`,
    },
    responseType: "stream",
    body: {
      stream: true,
      stream_options: {
        exclude_aggregated_audio: true, // no more summary block
      },
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

With this option, every chunk in the stream has `status: 1` and contains only an audio segment, except for the final chunk with `status: 2` that now has an empty audio field. This eliminates the need for the status-checking logic we previously had in `processEventData`.

## Replacing Manual SSE Parsing with eventsource-parser

In the previous implementation, we manually split the stream on `\n\n` boundaries and stripped `data: ` prefixes. While this worked for Minimax's specific formatting, it was fragile and made assumptions about the event structure. The [`eventsource-parser`](https://github.com/rexxars/eventsource-parser) library provides a spec-compliant SSE parser that handles all edge cases for us.

### Using EventSourceParserStream

The library exposes an [`EventSourceParserStream`](https://github.com/rexxars/eventsource-parser) -- a `TransformStream` that takes decoded text and outputs parsed SSE events. This is a direct replacement for our custom `TransformStream`:

```typescript
import { EventSourceParserStream } from "eventsource-parser/stream";

const eventStream = response
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new EventSourceParserStream());
```

Each event emitted by `EventSourceParserStream` has a `data` property containing the event payload (with the `data: ` prefix already stripped), an `event` property for the event type, and an `id` property for the event ID. For Minimax's API, we only need `event.data`.

### Simplified TransformStream

With the SSE parsing handled by the library, our `TransformStream` now only needs to extract audio data from parsed events:

```typescript
import { EventSourceParserStream } from "eventsource-parser/stream";

const audioTransform = new TransformStream({
  transform(event, controller) {
    try {
      const parsed = JSON.parse(event.data);

      if (parsed.base_resp?.status_code !== 0) {
        controller.error(parsed.base_resp?.status_msg || "Unknown API error");
        return;
      }

      if (parsed.data?.audio) {
        controller.enqueue(Buffer.from(parsed.data.audio, "hex"));
      }
    } catch (error) {
      // Skip malformed events
    }
  },
});

const audioStream = response
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(new EventSourceParserStream())
  .pipeThrough(audioTransform);

return sendStream(event, audioStream);
```

Compare this with the previous implementation: the buffer management, `\n\n` splitting, `data: ` prefix stripping, and `flush` logic are all gone. The `EventSourceParserStream` handles all of that correctly according to the SSE specification.

## Why Build a New SDK?

Minimax does provide an official JavaScript SDK, but it's an [MCP (Model Context Protocol) server](https://github.com/MiniMax-AI/MiniMax-MCP-JS) -- designed for AI agent tool-calling, not for direct use in application code. There is no official Node.js client library for calling the Minimax Speech API directly. If you want to integrate Minimax TTS into a server application, you're left writing raw HTTP calls yourself, which is what I was doing in the [previous post]({% post_url 2025-06-22-handling-minimax-tts-api-basic-and-streaming %}). That's the gap `minimax-speech-ts` fills: a proper typed client library for the HTTP API.

## Using minimax-speech-ts

While the above improvements already simplify the integration, there's still boilerplate involved: constructing the request, handling authentication, converting hex to buffers, and managing error codes. I built [`minimax-speech-ts`](https://github.com/williamchong/minimax-speech-ts) to wrap all of this into a clean TypeScript SDK.

### Installation

```bash
npm install minimax-speech-ts
```

### Basic Usage

```typescript
import { MiniMaxSpeech } from "minimax-speech-ts";

const client = new MiniMaxSpeech({
  apiKey: process.env.MINIMAX_API_KEY!,
  groupId: process.env.MINIMAX_GROUP_ID,
});

// Non-streaming: get complete audio as a Buffer
const result = await client.synthesize({
  text: "your text",
  model: "speech-02-hd",
  voiceSetting: {
    voiceId: "Chinese (Mandarin)_Warm_Bestie",
    speed: 0.95,
    pitch: -1,
    emotion: "neutral",
  },
  languageBoost: "Chinese,Yue",
});

await fs.promises.writeFile("output.mp3", result.audio);
```

### Streaming Usage

The SDK returns a `ReadableStream<Buffer>` that handles all the SSE parsing, hex decoding, and error handling internally:

```typescript
const stream = await client.synthesizeStream({
  text: "your text",
  voiceSetting: { voiceId: "Chinese (Mandarin)_Warm_Bestie" },
  streamOptions: { excludeAggregatedAudio: true },
});

// Use directly with sendStream in Nuxt/Nitro
return sendStream(event, stream);
```

The SDK uses a camelCase interface (`excludeAggregatedAudio`) and automatically converts to the snake_case wire format (`exclude_aggregated_audio`) expected by the API. It also provides typed error classes for different failure modes:

```typescript
import {
  MiniMaxAuthError,
  MiniMaxRateLimitError,
  MiniMaxValidationError,
} from "minimax-speech-ts";

try {
  const stream = await client.synthesizeStream({ text: "hello" });
} catch (e) {
  if (e instanceof MiniMaxRateLimitError) {
    // Back off and retry
  } else if (e instanceof MiniMaxAuthError) {
    // Invalid API key
  }
}
```

Beyond basic TTS, the SDK also covers voice cloning, voice design, async synthesis for long-form content, and voice management -- the full Minimax Speech HTTP API surface. WebSocket support is planned for a future release.

## Building a TypeScript SDK with Claude Code

The [`minimax-speech-ts`](https://github.com/williamchong/minimax-speech-ts) library was built entirely using [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code), Anthropic's CLI for Claude. The result is a ~1500-line TypeScript library with 79 tests, a single runtime dependency (`eventsource-parser`), dual ESM/CJS output, CI/CD, and TypeDoc-generated API documentation -- built in about a day.

I've been developing a mental framework for effective AI-assisted coding that I think of as three iterative steps: **Context**, **Limit**, and **Progress**. The development of this SDK is a good concrete example of how it plays out in practice.

### Step 1: Context -- Feed the AI the Right Information

The quality of AI-generated code is directly bounded by the context it has access to. My initial prompt to Claude Code referenced:

- The [Minimax API documentation](https://platform.minimax.io/docs/api-reference/speech-t2a-http) for the complete API spec
- The [MiniMax-MCP-JS](https://github.com/MiniMax-AI/MiniMax-MCP-JS) repository as a reference for how Minimax structures their APIs
- My [existing blog post]({% post_url 2025-06-22-handling-minimax-tts-api-basic-and-streaming %}) as context for the current "dumb" approach
- My other TypeScript libraries ([epubcheck-ts](https://github.com/williamchong/epubcheck-ts), [epub.ts](https://github.com/williamchong/epub.ts)) as style reference

The single most important factor was that Minimax provides their API documentation in markdown format (e.g. [`speech-t2a-http.md`](https://platform.minimax.io/docs/api-reference/speech-t2a-http.md)). Claude Code could fetch and read the full API specification directly, generating accurate type definitions, request/response handling, and validation logic without manual transcription. Pointing at my existing libraries meant it could match my preferences for project structure, tooling choices (tsup, vitest), and coding conventions without explicit configuration.

Without good context, the AI guesses. With good context, it builds.

### Step 2: Limit -- Set Boundaries and Constraints

Context alone produces code, but it doesn't guarantee *correct* code. The limit step is about defining and enforcing constraints that keep the output within acceptable bounds.

**Tests as limits**: Tests were written alongside the implementation from the very first feature commit. This meant Claude Code could run the 79 tests after each change and catch regressions immediately, covering all API methods, error classification, snake_case mapping, and streaming edge cases. Tests are the most concrete form of limit -- they define exactly what "correct" means.

**Linting as limits**: Adding ESLint with `typescript-eslint` in strict mode (including `no-explicit-any`) prevented the AI from taking shortcuts with loose types. TypeScript's strict mode itself is a limit -- it forces the generated code to handle nullability and type narrowing properly.

**Validation as limits**: Client-side parameter validation (emotion/model compatibility, WAV format restrictions, required field checks) encodes domain-specific constraints that the AI learned from the API docs. The declarative `validate()` helper using `[condition, message]` tuples made these constraints explicit and testable.

### Step 3: Progress -- Review, Fix, and Ship

With context and limits in place, the final step is iterating toward completion: expanding scope, catching remaining issues, and preparing for release.

**Expanding scope**: Starting from core `synthesize()` and `synthesizeStream()` methods, I asked Claude Code to expand to the full API surface -- async synthesis, file upload, voice cloning, voice design, voice management. The limits from step 2 ensured each expansion didn't break existing functionality.

**`/review` for catching subtle bugs**: Claude Code has a built-in `/review` command that performs a code review on the current codebase. Running `/review` after the main implementation caught several issues: a TypeScript overload ordering bug (where the general signature shadowed the specific `outputFormat: 'url'` overload), missing `traceId` propagation in error paths, and the need to URL-encode the `groupId` query parameter. These are exactly the kinds of subtle issues that slip through during development but get caught by a systematic review pass.

**`AGENTS.md` and `.github/copilot-instructions.md` for ongoing progress**: The library includes an [`AGENTS.md`](https://github.com/williamchong/minimax-speech-ts/blob/master/AGENTS.md) file that documents the architecture, key patterns, test patterns, and build commands. This file is symlinked as `CLAUDE.md` so that Claude Code automatically picks it up as project instructions, while other AI coding tools (GitHub Copilot, Cursor, etc.) can read it via `AGENTS.md` -- the emerging convention for AI agent context files. The symlink trick avoids maintaining two copies of the same content. The repository also has a [`.github/copilot-instructions.md`](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions) file, generated by GitHub Copilot agent itself based on the existing codebase and `AGENTS.md`. GitHub Copilot automatically incorporates these instructions when performing [code reviews on pull requests](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions), so its PR reviews are aware of the project's specific patterns (e.g. "validate before fetch", "camelCase public API with snake_case wire format") rather than applying generic heuristics. These files feed back into step 1 -- they become context for future AI-assisted development, creating a virtuous cycle.

**Shipping**: Finally, Claude Code generated the README, CI workflow (Node 18/20/22), TypeDoc configuration, license, and npm metadata. Throughout the process, it maintained consistency in patterns like the camelCase-to-snake_case conversion and the error hierarchy.

### The Takeaway

The **Context → Limit → Progress** cycle isn't a one-shot sequence -- it's iterative. Each round of progress reveals new context needs and new constraints to enforce. The entire library went from empty directory to published npm package in a day, but the quality came from deliberately feeding the right context, setting clear limits, and iterating through review. I'll explore this framework in more depth in a future post.

## Conclusion

The three improvements covered in this post each address a different layer of the integration:

- **`exclude_aggregated_audio`** eliminates the need to filter the summary block at the API layer
- **`eventsource-parser`** replaces manual SSE parsing with a spec-compliant library
- **`minimax-speech-ts`** wraps the entire API into a typed SDK with streaming, error handling, and camelCase ergonomics

If you're integrating Minimax TTS into a Node.js application, using the SDK is the most straightforward path. If you need more control or are integrating with a different runtime, the combination of `exclude_aggregated_audio` and `eventsource-parser` provides a clean foundation.

## Additional Resources

### Related Articles

- [Handling Minimax TTS API: Basic HTTP and Streaming](/2025/06/22/handling-minimax-tts-api-basic-and-streaming/) (Previous post)
- [Convert Google text to speech API result to HTTP streamed response](/2023/10/13/convert-google-text-to-speech-to-nodejs-stream/)
- [Convert Azure text to speech API result to HTTP streamed response](/2023/10/15/convert-azure-text-to-speech-to-nodejs-stream/)
- [Convert Azure text to speech API result to a web ReadableStream](/2025/09/02/convert-azure-text-to-speech-to-web-readable-stream/)
- [Convert AWS Polly text to speech API result to HTTP streamed response](/2025/06/06/convert-aws-polly-to-nodejs-stream/)

### Links

- [minimax-speech-ts on GitHub](https://github.com/williamchong/minimax-speech-ts)
- [minimax-speech-ts on npm](https://www.npmjs.com/package/minimax-speech-ts)
- [minimax-speech-ts API Reference](https://williamchong.github.io/minimax-speech-ts/)
- [eventsource-parser on GitHub](https://github.com/rexxars/eventsource-parser)
- [Minimax TTS API Documentation](https://platform.minimax.io/docs/api-reference/speech-t2a-http)
- [Claude Code](https://docs.anthropic.com/en/docs/build-with-claude/claude-code)

---

*By the way -- guess what tool and process I used to write this blog post.*
