# Streaming Protocol

`POST /agents/research/stream` answers with Server-Sent Events. The answer arrives token by token, and alongside it the agent reports what it is doing — which legal areas it picked, which searches it dispatched, what each search found — so you can show progress instead of a spinner.

This page is the complete event catalogue plus a worked parser.

---

## Table of contents

1. [Transport](#transport)
2. [Frame format](#frame-format)
3. [Event order](#event-order)
4. [Text events](#text-events)
5. [Citations](#citations)
6. [Progress events](#progress-events)
7. [Follow-up queries](#follow-up-queries)
8. [Feedback](#feedback)
9. [Errors](#errors)
10. [Parsing the stream](#parsing-the-stream)
11. [Using the AI SDK](#using-the-ai-sdk)

---

## Transport

```http
POST https://genial-api.sdu.nl/v10/agents/research/stream
Authorization: Bearer eyJraWQiOi...
X-API-Tenant-Id: your-tenant-id
Content-Type: application/json
```

The request body is identical to the non-streaming endpoint — see [Research agent](02-research-agent.md#request-body).

Response headers:

```http
200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no
x-vercel-ai-ui-message-stream: v1
```

The stream is the **AI SDK UI Message Stream**, version 1. If your frontend already uses the Vercel AI SDK, point it at this endpoint and the parsing is done for you — see [Using the AI SDK](#using-the-ai-sdk). Everyone else can parse it with any SSE client.

Disable response buffering on any proxy in front of this route. `X-Accel-Buffering: no` covers nginx; other proxies need their own setting. A buffering proxy turns a live stream into one long pause followed by everything at once.

## Frame format

Every frame is a `data:` line carrying one JSON object, followed by a blank line:

```plaintext
data: {"type":"text-delta","id":"txt_9c1f…","delta":"Bij een huurovereenkomst"}

```

There are no `event:` names, no `id:` lines and no retry directives. The frame's meaning is in the JSON `type` field.

The stream ends with a literal sentinel that is **not** JSON:

```plaintext
data: [DONE]

```

Check for `[DONE]` before attempting to parse a frame, or your JSON parser will throw on the last one.

## Event order

```plaintext
start
text-start
  ├─ data-abusive-check
  ├─ data-filter-messages          (only when there is conversation history)
  ├─ data-planner
  ├─ data-fan-out
  ├─ data-legislation | data-case-law | data-commentary
  │  | data-practice-notes | data-other-sources   (in any order, repeatable)
  ├─ data-verify
  ├─ (planner → fan-out → searches → verify again, if the agent replans)
  ├─ data-template-selector
  ├─ text-delta                    (many; the answer)
  ├─ data-reference                (interleaved with text-delta, as documents are cited)
  └─ data-followup-queries
data-feedback
[data-error]                       (only when something went wrong)
text-end
finish
[DONE]
```

Guarantees worth building on:

- `start` is always first, `text-start` always second.
- `text-end`, `finish` and `[DONE]` always arrive, in that order, even when the turn failed.
- `text-delta` frames are only emitted between `text-start` and `text-end`.
- Progress events are best-effort ordering. Corpus searches run in parallel, so their events interleave and their relative order is not fixed. Do not key logic on which arrives first.
- The set of progress events varies per turn. A question answered from history alone may emit no search events at all.

## Text events

### `start`

```json
{ "type": "start", "messageId": "msg_4a7f2c1e9b8d4f6a" }
```

Opens the assistant message. `messageId` is generated per turn; use it as the id of the message you store.

### `text-start`

```json
{ "type": "text-start", "id": "txt_2e8b6d1a4c7f9035" }
```

Opens the text block. Every subsequent `text-delta` carries this same `id`. There is exactly one text block per turn.

### `text-delta`

```json
{ "type": "text-delta", "id": "txt_2e8b6d1a4c7f9035", "delta": "Bij een huurovereenkomst " }
```

Append `delta` to the answer. Deltas are not aligned to word or sentence boundaries — concatenate them verbatim and never trim.

The concatenation of all deltas is Dutch Markdown. Render it as Markdown once the block closes, or incrementally with a streaming-tolerant Markdown renderer.

### `text-end`

```json
{ "type": "text-end", "id": "txt_2e8b6d1a4c7f9035" }
```

Closes the text block. No more deltas follow.

### `finish`

```json
{ "type": "finish" }
```

The turn is complete. `[DONE]` follows immediately.

## Citations

The agent cites a **passage** of a document, not the document as a whole. Each citation reaches you as up to two frames.

### `data-reference`

```json
{
  "type": "data-reference",
  "id": "BWBR0005290",
  "data": {
    "document_id": "BWBR0005290",
    "display_title": "Burgerlijk Wetboek Boek 7",
    "original_source_url": "https://…",
    "passages": [{ "identifier": "p-BWBR0005290-7-2", "text": "De huurovereenkomst eindigt…" }],
    "published": [],
    "about": []
  }
}
```

`id` is the document id. `data` is a full [`SearchResult`](05-schemas.md#searchresult), in `snake_case`.

**Each document is sent once**, the first time it is cited. Later citations of the same document arrive as the marker alone. Keep the documents you have received in a map keyed by `id`.

### The inline marker

Immediately after the reference (or on its own, for a repeat citation), a `text-delta` carries the marker:

```json
{ "type": "text-delta", "id": "txt_2e8b6d1a4c7f9035", "delta": "<sup>p-BWBR0005290-7-2</sup>" }
```

The value inside `<sup>` is a **passage identifier**, not a document id.

### Resolving a marker

1. Build a passage index: for every `data-reference` received, map each `passages[].identifier` to that document.
2. When rendering the Markdown, replace each `<sup>…</sup>` with a footnote whose target is the document the index resolves the identifier to.
3. `passage.text` is the exact sentence the claim rests on — good hover content.

Number footnotes by first appearance in the text, not by the order references arrived; a document is often sent before the sentence that cites it is finished.

A marker whose identifier resolves to nothing should be dropped rather than displayed. It means a malformed citation slipped through.

## Progress events

Every progress frame has the shape:

```json
{ "type": "data-<step>", "data": {} }
```

Some carry an `id`. **An `id` names the item being reported on, not the event.** Two frames sharing an `id` are the same item revised — replace, do not append. Frames without an `id` stand alone.

### `data-abusive-check`

```json
{ "type": "data-abusive-check", "data": { "query_type": "normal" } }
```

`query_type` is `"normal"` or `"abusive"`. An abusive query is answered with a refusal and no research is done.

### `data-filter-messages`

```json
{ "type": "data-filter-messages", "data": { "message_count": 4 } }
```

How many messages of conversation history were judged relevant and kept. Only emitted when you sent a `chatId`.

### `data-planner`

```json
{
  "type": "data-planner",
  "data": {
    "legal_areas": ["vastgoed- en huurrecht"],
    "execution_plans": [
      {
        "legal_issue": "opzegtermijn huurovereenkomst bepaalde tijd",
        "agent": "legislation",
        "legal_area_facet": ["vast"],
        "retrieval_mode": null
      }
    ]
  }
}
```

The research strategy: the legal areas the question was classified under, and one [`ExecutionPlan`](05-schemas.md#executionplan) per search about to run. Emitted again if the agent replans.

### `data-fan-out`

```json
{ "type": "data-fan-out", "data": { "dispatched": [{ "routes": ["legislation", "case_law"] }] } }
```

Which searches were actually dispatched. Useful for showing "searching legislation and case law…" before any results exist.

### Search events

`data-legislation`, `data-case-law`, `data-commentary`, `data-practice-notes`, and `data-other-sources` all share one shape:

```json
{
  "type": "data-case-law",
  "data": {
    "legal_issue": "opzegging huurovereenkomst bepaalde tijd",
    "results": [
      {
        "document_id": "ECLI_NL_HR_2021_1234",
        "title": "Hoge Raad, 03-09-2021, ECLI:NL:HR:2021:1234",
        "document_date": "2021-09-03",
        "author": null,
        "findings": [
          {
            "finding": "De Hoge Raad oordeelt dat…",
            "relevance": "critical",
            "passage_id": "p-ECLI_NL_HR_2021_1234-12"
          }
        ]
      }
    ]
  }
}
```

`legal_issue` is the query that search ran; `results` is a [`DistilledSource[]`](05-schemas.md#distilledsource) — the documents found, reduced to the findings that bear on the question.

These arrive as each search finishes, so they interleave. `data-document` reports on attached documents rather than a corpus. Any of them can repeat if the agent replans.

An empty `results` array means that search found nothing. It is not an error on its own — another search may still carry the answer.

### `data-verify`

```json
{
  "type": "data-verify",
  "data": {
    "sufficient": false,
    "rationale": "de opzegtermijn bij een huurovereenkomst voor bepaalde tijd is niet onderzocht"
  }
}
```

The agent's own judgement on whether it has enough to answer. `sufficient: false` means it is about to plan another round — expect a second `data-planner`, more searches and another `data-verify`.

### `data-template-selector`

```json
{ "type": "data-template-selector", "data": { "answer_template": "cirac" } }
```

The shape the answer will take, chosen from the question: `cirac`, `memo`, `email`, `list_sources`, `simple` or `transform`. Emitted just before the text starts.

## Follow-up queries

```json
{ "type": "data-followup-queries", "data": ["Wat gebeurt er bij stilzwijgende verlenging?"] }
```

Suggested next questions in Dutch, derived from what was found. Offer them as one-click prompts. Sent once, near the end; may be absent.

## Feedback

```json
{
  "type": "data-feedback",
  "data": { "trace_id": "tr-8f2c…", "submitted": false, "sentiment": "", "rationale": "" }
}
```

`trace_id` identifies this turn in the platform's tracing backend. Store it with the message: it is what a thumbs-up/thumbs-down on that answer is attached to, and what support needs to look a specific answer up. Sent once per turn, after the answer.

## Errors

```json
{
  "type": "data-error",
  "data": {
    "message": "No search results found for the user's query.",
    "user_message": "We konden geen relevante documenten vinden voor je vraag. Probeer je vraag te herformuleren of opnieuw te stellen.",
    "details": { "document_id": "…" }
  }
}
```

Three things to know:

1. **The HTTP status is still `200`.** The connection was fine; the turn was not.
2. **The stream still closes cleanly.** `text-end`, `finish` and `[DONE]` follow as normal.
3. **This is not the AI SDK's `error` event.** It is a data event named `data-error`, so a generic AI SDK client will treat it as an ordinary custom data part rather than an error. Handle it explicitly.

When a `data-error` arrives, the answer text is either absent or incomplete. Show `user_message` to the user; log `message` and `details`.

The full list of in-band errors is in [Errors](01-overview.md#errors).

## Parsing the stream using the AI SDK

The stream is emitted in the Vercel AI SDK's UI Message Stream format, so `useChat` consumes it without a custom parser:

```ts
const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: `${BASE_URL}/agents/research/stream`,
    headers: {
      Authorization: `Bearer ${token}`,
      "X-API-Tenant-Id": tenantId,
    },
  }),
});
```

Message parts then arrive as `text`, `data-reference`, `data-planner`, `data-legislation` and so on, matching the events above one for one.

Two things the SDK will not do for you:

- **Citations.** `<sup>…</sup>` markers reach your renderer as literal text. Resolve them against the `data-reference` parts yourself, as shown above.
- **Errors.** `data-error` is a data part, not the SDK's `error` event, so `onError` never fires for it. Check the parts.

## Parsing the stream (custom)

The rules, in order:

1. Split the stream on blank lines to get frames.
2. Strip the `data: ` prefix.
3. If what remains is `[DONE]`, stop.
4. Parse the rest as JSON and switch on `type`.
5. Ignore unknown `type` values. New event types are added without a breaking change, and a client that throws on the unfamiliar will break on the next release.

A minimal TypeScript reader:

```ts
type Frame = { type: string; [key: string]: unknown };

async function* readFrames(response: Response): AsyncGenerator<Frame> {
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });

    // Frames are separated by a blank line; the last piece may be incomplete.
    const chunks = buffer.split("\n\n");
    buffer = chunks.pop() ?? "";

    for (const chunk of chunks) {
      const line = chunk.trim();
      if (!line.startsWith("data:")) continue;
      const payload = line.slice(5).trim();
      if (payload === "[DONE]") return;
      yield JSON.parse(payload) as Frame;
    }
  }
}
```

And a consumer that assembles a renderable answer:

```ts
const BASE_URL = "https://genial-api.sdu.nl/v10";

type Passage = { identifier: string; text: string };
type Reference = { document_id: string; display_title?: string; passages?: Passage[] };

async function research(body: unknown, token: string, tenantId: string) {
  const response = await fetch(`${BASE_URL}/agents/research/stream`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "X-API-Tenant-Id": tenantId,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (!response.ok) throw new Error(`Request failed: ${response.status}`);

  let text = "";
  const references = new Map<string, Reference>();
  const passageToDocument = new Map<string, string>();
  const progress: Frame[] = [];
  let followups: string[] = [];
  let traceId: string | undefined;
  let error: { message: string; user_message: string } | undefined;

  for await (const frame of readFrames(response)) {
    switch (frame.type) {
      case "text-delta":
        text += frame.delta as string;
        break;

      case "data-reference": {
        const reference = frame.data as Reference;
        references.set(frame.id as string, reference);
        for (const passage of reference.passages ?? []) {
          passageToDocument.set(passage.identifier, reference.document_id);
        }
        break;
      }

      case "data-followup-queries":
        followups = frame.data as string[];
        break;

      case "data-feedback":
        traceId = (frame.data as { trace_id: string }).trace_id;
        break;

      case "data-error":
        error = frame.data as typeof error;
        break;

      case "start":
      case "text-start":
      case "text-end":
      case "finish":
        break;

      default:
        // Every remaining data-* frame is a progress update.
        if (frame.type.startsWith("data-")) progress.push(frame);
        break;
    }
  }

  return { text, references, passageToDocument, progress, followups, traceId, error };
}
```

Resolving the citations afterwards is a single pass over the text:

```ts
const CITATION = /<sup>([^<]+)<\/sup>/g;

function numberCitations(text: string, passageToDocument: Map<string, string>) {
  const numbers = new Map<string, number>();

  const rendered = text.replace(CITATION, (whole, passageId: string) => {
    const documentId = passageToDocument.get(passageId);
    if (!documentId) return ""; // unresolved marker: drop it
    if (!numbers.has(passageId)) numbers.set(passageId, numbers.size + 1);
    return `[^${numbers.get(passageId)}]`;
  });

  return { rendered, numbers };
}
```
