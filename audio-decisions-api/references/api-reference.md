# API reference summary

**Written in November 2026 for the first release of the API.** The [API reference](https://dev.hume.ai/audio-decisions/reference) is the source of truth: check it whenever this page and the reference disagree.

## Endpoints

| Endpoint | What it does |
| --- | --- |
| `POST https://api.cloud.hume.ai/v0/decisions/file` | Answers questions about one audio file |
| `POST https://api.cloud.hume.ai/v0/decisions/file/input_tokens` | Counts the input tokens a decisions request would be billed for, without sending audio |
| `GET https://api.cloud.hume.ai/v0/decisions/run` | Lists your organization's runs, newest first |
| `GET https://api.cloud.hume.ai/v0/decisions/run/{run_id}` | Gets one run |

**Authentication:** your API key in the `X-Hume-Api-Key` header. The SDKs read it from the `HUME_API_KEY` environment variable. Keep API keys on a server: to use the API from a web page, relay requests through your server.

## Decisions request

The body is `multipart/form-data` with two parts: `request` (JSON: the questions and optional context) and `file` (one MP3 or WAV file).

```bash
curl https://api.cloud.hume.ai/v0/decisions/file \
  -H "X-Hume-Api-Key: $HUME_API_KEY" \
  -F 'request=<request.json;type=application/json' \
  -F file=@call.wav
```

`request.json`:

```json
{
  "context": "Customer support call. This is the caller's channel only.",
  "questions": {
    "frustrated": {
      "type": "noul",
      "instructions": "Does the caller sound frustrated?",
      "criteria": {
        "true": "Frustration is audible in the caller's voice",
        "false": "The caller sounds calm, or only says in words that they're frustrated"
      }
    },
    "reason": {
      "type": "choice",
      "instructions": "What is the caller asking for?",
      "criteria": [
        { "name": "billing", "description": "A charge, refund or invoice" },
        { "name": "technical", "description": "Something isn't working" },
        { "name": "cancel", "description": "Ending their service" },
        { "name": "other" }
      ]
    },
    "pace": {
      "type": "score",
      "instructions": "How fast does the caller speak?",
      "criteria": ["Slow", "Moderate", "Fast"]
    },
    "laughs": { "type": "moment", "instructions": "When does the caller laugh?" }
  }
}
```

A `noul` question's `criteria` is optional. Unknown fields in the request are rejected.

## Response

```json
{
  "run_id": "019a1f41-2b6c-7e05-8a3d-71c9e0b54f26",
  "answers": {
    "frustrated": { "type": "noul", "noul": 0.8123 },
    "reason": {
      "type": "choice", "choice": "billing", "confidence": 0.41,
      "probabilities": { "billing": 0.56, "technical": 0.21, "cancel": 0.15, "other": 0.08 }
    },
    "pace": {
      "type": "score", "score": 1.42, "confidence": 0.33,
      "probabilities": { "0": 0.12, "1": 0.34, "2": 0.54 },
      "legend": { "0": "Slow", "1": "Moderate", "2": "Fast" }
    },
    "laughs": {
      "type": "moment",
      "moments": [ { "audio_start_ms": 12340, "audio_end_ms": 13910, "probability": 0.77 } ]
    }
  }
}
```

Values are illustrative.

- **`noul`** is the share of raters who would answer yes.
- **`choice`** and **`score`** probabilities sum to 1. `score` is the mean level, counted from 0. `probabilities` and `legend` for `score` are keyed by level index as strings, so don't rely on key order.
- **`confidence`** runs from 0 (every option equally likely) to 1 (all on one option). It measures how much raters would agree, not how sure the model is.
- **`moments`** are in milliseconds from the start of the file, in order of `audio_start_ms`, with overlapping candidates merged. An empty list means the event was not found. A `moment` answer gives times but no share of raters, so to ask whether an event occurs as well as when, add a `noul` question.
- **`run_id`** identifies the run. The run endpoints return a run's status, when it started and ended, and the request it evaluated, not its answers: store the answers yourself.
- **Ignore unknown fields and unknown answer types** rather than treating them as errors. The API adds fields as it grows.

## Counting input tokens

Requests are billed in input tokens, counted from the context, the questions and the duration of the audio. The input tokens endpoint takes a JSON body with `audio_duration_ms`, `context` and `questions`, and returns `input_tokens`. It takes no audio, so you can count a request before sending it. A duration under 5 seconds or over 5 minutes is refused the same way a decisions request would be, so count a long recording one chunk at a time: each chunk is billed as its own request.

## SDKs

**Python** (3.10 or later):

```bash
pip install hume-audio-decisions
```

```python
from hume_audio_decisions import (
    Answer_Noul, Criterion, DecisionsClient, DecisionsRequest,
    Question_Choice, Question_Noul,
)
from hume_audio_decisions.core import ApiError

client = DecisionsClient()  # reads HUME_API_KEY

request = DecisionsRequest(
    context="Customer support call. This is the caller's channel only.",
    questions={
        "frustrated": Question_Noul(instructions="Does the caller sound frustrated?"),
        "tone": Question_Choice(
            instructions="Which best describes the caller's tone?",
            criteria=[Criterion(name="calm"), Criterion(name="tense"), Criterion(name="angry")],
        ),
    },
)

with open("caller.wav", "rb") as audio:
    response = client.decisions.create(file=("caller.wav", audio, "audio/wav"), request=request)

answer = response.answers["frustrated"]
if isinstance(answer, Answer_Noul):
    print(answer.noul)

count = client.decisions.input_tokens(
    audio_duration_ms=135_000, context=request.context, questions=request.questions
)
```

Use `"audio/mpeg"` for MP3. Failed requests raise `ApiError` with `status_code` and `body`. `AsyncDecisionsClient` has the same interface with `async` methods, for sending requests for several recordings concurrently.

**TypeScript** (Node.js 18 or later):

```bash
npm install @humeai/audio-decisions
```

```typescript
import { DecisionsClient, DecisionsError } from "@humeai/audio-decisions";

const client = new DecisionsClient(); // reads HUME_API_KEY

const response = await client.decisions.create({
  file: { path: "caller.wav", contentType: "audio/wav" },
  request: {
    context: "Customer support call. This is the caller's channel only.",
    questions: {
      frustrated: { type: "noul", instructions: "Does the caller sound frustrated?" },
    },
  },
});

const answer = response.answers.frustrated;
if (answer.type === "noul") console.log(answer.noul);
```

The TypeScript SDK uses camelCase fields (`runId`, `audioStartMs`, `audioEndMs`, `inputTokens`). Request and response types are exported under the `Decisions` namespace. Failed requests throw `DecisionsError` with `statusCode` and `body`.

**Retries and timeouts:** both SDKs retry a request that fails with a 408, 429 or 5xx status up to 2 times, waiting as long as `Retry-After` asks; the Python SDK also retries 409. Pass `max_retries` (`maxRetries`) to the client, or in the request options for one request, to change the limit. Requests time out after 60 seconds unless you pass a different `timeout` (`timeoutInSeconds`).

### Audio the SDKs prepare

The API resamples every file to 16 kHz and mixes its channels into one before the model hears it. The SDKs do the same to a WAV file before uploading it, which keeps uploads small. `downmix_strategy` (`downmixStrategy`) sets how:

1. `"mix"`, the default, resamples to 16 kHz and mixes two channels into one the way the API does.
2. `"first_channel"` keeps only the first channel. Use it to judge the speaker on the first channel of a stereo call. For any other channel, extract it yourself and send it as its own file.
3. `None` in Python, `"none"` in TypeScript, sends the file as it is.

MP3 files, and WAV files the SDK cannot read, are always sent as they are.

### Long recordings

`decisions.create_chunked` (`createChunked`) cuts a recording into chunks no longer than `max_chunk_ms` (`maxChunkMs`) and sends each chunk in its own request, one at a time and in order, with the same questions and context. Each chunk gets its own response, `run_id` and bill. Nothing combines the answers across chunks: you do.

```python
with open("long_call.wav", "rb") as audio:
    result = client.decisions.create_chunked(
        file=("long_call.wav", audio, "audio/wav"), request=request, max_chunk_ms=300_000
    )

for chunk in result.chunks:
    if chunk.response is None:
        print(f"chunk {chunk.index}: {chunk.status}, {chunk.error}")
        continue
    print(chunk.start_ms, chunk.end_ms, chunk.response.answers["frustrated"])
```

- **`max_chunk_ms` has no default.** 300000 is the longest chunk the API accepts and sends the fewest requests. Shorter chunks give answers about shorter stretches.
- **Chunks are equal in length by default,** so the last one isn't a short remainder. `split="fixed"` makes every chunk `max_chunk_ms` long except the last.
- **Moment times are measured from the start of the chunk.** Add the chunk's `start_ms` to place a moment in the recording.
- **`start_ms` and `end_ms`** send only part of the recording.
- **A failed chunk doesn't raise.** Its error is recorded on the chunk, so check `result.complete` before relying on the result. To resume a job that stopped, call `create_chunked` again with `start_ms=result.resume_ms` and the same `end_ms`.
- **`decisions.iter_create_chunked`** (`iterCreateChunked`) yields each chunk's result as soon as it's done, so you can save each response before the next chunk is sent.
- **`iter_audio_chunks`** (`iterAudioChunks`) only cuts the recording, for sending chunks some other way, such as several at a time with the async client. It's in `hume_audio_decisions.audio_helpers` and `@humeai/audio-decisions/audio-helpers`. Chunks it yields are already mixed down, so send them with `downmix_strategy=None` (`"none"`).

Write the context and questions to hold for any chunk: the model answers about each chunk as if it were the whole recording, so "the agent speaks first" is true only of the first chunk.

### Runs

Every successful decisions request is recorded as a run. `runs.list` returns your organization's runs a page at a time, filtered by `user_id`, `status`, `started_after` and `started_before`; pass a page's `next_cursor` back as `cursor` for the next page. `runs.get` returns one run by `run_id`. Neither returns answers.

## Errors

Error bodies are `{ "type": "error", "code", "message", "param"?, "request_id" }`. `param` points to the field the error concerns, when there is one. Quote the `request_id` (also in the `X-Request-Id` header) when contacting [support](https://dev.hume.ai/audio-decisions/docs/support).

| Status | Code | Cause |
| --- | --- | --- |
| 400 | `message_invalid` | Missing `request` part, zero or several `file` parts, or JSON that doesn't match the schema outside a question |
| 400 | `question_invalid` | A question doesn't match the schema, for example duplicate option names, or fewer than two options |
| 400 | `audio_decode_failed` | The audio file couldn't be decoded |
| 400 | `audio_format_unsupported` | Not MP3 or WAV |
| 400 | `audio_too_short` | Under 5 seconds |
| 400 | `audio_too_long` | Over 5 minutes |
| 400 | `invalid_request` | A query parameter or `cursor` the run list doesn't accept |
| 401 | | Missing or rejected API key. Answered by Hume's API gateway, so the body has no `code` |
| 402 | `payment_required` | No payment method on file, or an unpaid invoice |
| 403 | `forbidden` | The key doesn't grant access to this API, or the organization isn't permitted to use it |
| 404 | `not_found` | No run with this ID in your organization |
| 413 | `payload_too_large` | The `request` and `file` parts together are over 25,000,000 bytes |
| 415 | `unsupported_media_type` | The body isn't `multipart/form-data` (or `application/json` for input tokens) |
| 429 | | A rate limit. Answered by the gateway with no `code`. Wait for `Retry-After` seconds, then retry |
| 500 | `internal_error` | Retry with backoff; contact support with the `request_id` if it persists |
| 503 | `server_busy` | The service is at capacity. Wait for `Retry-After` seconds, then retry |

## Limits

- **One file per request,** MP3 or WAV, **from 5 seconds to 5 minutes**.
- **Request size:** up to 25,000,000 bytes for the `request` and `file` parts together. With the SDKs' default `downmix_strategy`, 5 minutes of mono or stereo WAV always fits.
- **Questions:** up to 100 per request, with IDs of up to 256 characters. `choice` and `score` questions take 2 to 100 options or levels.
- **Text length** of `context` and instructions is capped. Check the reference for the current values.
- **Rate limits** depend on your plan.

## Behaviour to design around

- **Questions in a request are answered in one model inference.** Adding, removing or reordering questions can change answers, so keep the same set in the same order for recordings you'll compare.
- **The model can be updated without an API version change.** Keep a labelled check set and re-run it after updates.
- **No memory between requests.** Each request is judged on its own audio and context.

## Not in the first release, and what to do meanwhile

| Not yet available | Do this meanwhile |
| --- | --- |
| Streaming or real-time endpoint | Send rolling windows of recent audio as ordinary requests ([audio-units.md](audio-units.md#real-time)) |
| Several files in one request, comparing two clips directly | Ask the same questions of each clip in separate requests and compare the answers in your code |
| Batch endpoint | One request per file, several in parallel within your rate limit; `create_chunked` for one long recording |
| Audio by URL | Download the file and upload it |
| Choosing a channel or time range in the request | Use the SDK: `downmix_strategy="first_channel"` keeps the first channel of a WAV, and `create_chunked` takes `start_ms` and `end_ms`. Otherwise cut the channel or range yourself ([audio-units.md](audio-units.md#preparing-audio)) |
