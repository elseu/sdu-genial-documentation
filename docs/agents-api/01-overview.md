# GenIA-L Agents API

The Agents API answers Dutch legal questions over the Sdu corpus. It exposes two kinds of endpoint:

- **The research agent** — plans a research strategy, searches every relevant corpus, reads what it finds and writes a cited answer. Available streaming (SSE) and non-streaming (JSON).
- **The corpus agents** — a single corpus searched directly. No planning, no prose: you get structured findings with the passages they came from.

If you want an answer for an end user, call the research agent. If you want search results to render or post-process yourself, call a corpus agent.

---

## Table of contents

1. [Base URL and versioning](#base-url-and-versioning)
2. [Endpoints](#endpoints)
3. [Request headers](#request-headers)
4. [Naming conventions](#naming-conventions)
5. [Rate limiting](#rate-limiting)
6. [Errors](#errors)
7. [Timeouts](#timeouts)
8. [Where to go next](#where-to-go-next)

---

## Base URL and versioning

```plaintext
https://genial-api.sdu.nl/{VERSION}/agents/...
```

- **`{VERSION}`**: the API version, **`v10` or later**. The agent endpoints do not exist before v10.
- Every path in this documentation is written relative to that prefix. The examples use `v10`; substitute the version you were onboarded to.

## Endpoints

| Method | Path                            | Answers with     | Documented in                                                                    |
| ------ | ------------------------------- | ---------------- | -------------------------------------------------------------------------------- |
| `POST` | `/agents/research/stream`       | SSE event stream | [Research agent](02-research-agent.md), [Streaming protocol](03-streaming-protocol.md) |
| `POST` | `/agents/research`              | JSON             | [Research agent](02-research-agent.md)                                              |
| `POST` | `/agents/search/legislation`    | JSON             | [Corpus agents](04-corpus-agents.md)                                                |
| `POST` | `/agents/search/case_law`       | JSON             | [Corpus agents](04-corpus-agents.md)                                                |
| `POST` | `/agents/search/commentary`     | JSON             | [Corpus agents](04-corpus-agents.md)                                                |
| `POST` | `/agents/search/practice_notes` | JSON             | [Corpus agents](04-corpus-agents.md)                                                |
| `POST` | `/agents/search/other_sources`  | JSON             | [Corpus agents](04-corpus-agents.md)                                                |

Every endpoint is `POST` with a JSON body, and every endpoint requires authentication.

## Request headers

| Header            | Required | Value                                                             |
| ----------------- | -------- | ----------------------------------------------------------------- |
| `Authorization`   | Yes      | `Bearer <access_token>` — see [Authentication](../authentication.md) |
| `X-API-Tenant-Id` | Yes      | Your tenant identifier — see [Authentication](../authentication.md)  |
| `Content-Type`    | Yes      | `application/json`                                                |

## Naming conventions

**Requests accept both** `camelCase` and `snake_case` for envelope fields. `{"legalIssue": "..."}` and `{"legal_issue": "..."}` are the same request. Message parts follow the camelCase spelling used by the AI SDK (`sourceId`, `mediaType`, `providerMetadata`).

**Responses are always `snake_case`.** This includes the objects nested inside streamed `data-reference` events. A response never echoes the camelCase spelling you sent. Do not build a client that assumes symmetry here — it is the single most common integration mistake against this API.

## Rate limiting

The service enforces four rolling windows at once:

| Window | Limit |
| ------ | ----- |
| second | 5     |
| minute | 20    |
| hour   | 200   |
| day    | 1000  |

Exceeding any window returns `429 Too Many Requests` with a JSON body:

```json
{ "detail": "Too Many Requests" }
```

These are the service's own limits. Your gateway usage plan (TRIAL, BASIC, or a contracted plan) may impose stricter quotas on top of them — see [Authentication](../authentication.md#access-levels). Whichever limit is lower is the one you will hit.

A research turn fans out into many internal searches, but it counts as **one** request against these limits.

## Errors

### Transport errors

Returned as a normal HTTP error, before any agent work starts.

| Status | Meaning                                                            |
| ------ | ------------------------------------------------------------------ |
| `401`  | Missing, malformed or expired access token                         |
| `403`  | The token is valid but lacks the permission this endpoint requires |
| `422`  | The request body did not validate — see the shape below            |
| `429`  | Rate limit exceeded                                                |
| `5xx`  | The service or one of its dependencies failed                      |

A `422` body follows the FastAPI convention, naming the offending field:

```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "message"],
      "msg": "Field required"
    }
  ]
}
```

### In-band errors

An agent can fail **after** it has already started answering — no search results, an unreadable attachment, an upstream model error. Those do not change the HTTP status, which stays `200`. They arrive in the response body instead:

- **Streaming**: as a `data-error` event, just before the stream closes normally. See [Streaming protocol](03-streaming-protocol.md#errors).
- **Non-streaming**: in the `errors` array of the response body.

Both carry the same object:

```json
{
  "message": "No search results found for the user's query.",
  "user_message": "We konden geen relevante documenten vinden voor je vraag. Probeer je vraag te herformuleren of opnieuw te stellen.",
  "details": { "document_id": "..." }
}
```

| Field          | Type             | Description                                                      |
| -------------- | ---------------- | ---------------------------------------------------------------- |
| `message`      | `string`         | English, for your logs. Not written for end users.               |
| `user_message` | `string`         | Dutch, written to be shown to an end user as-is.                 |
| `details`      | `object \| null` | Optional string-to-string context. Present only for some errors. |

**A `200` response is not proof of success.** Check `errors` (JSON) or watch for `data-error` (SSE) on every call.

## Timeouts

A research turn plans, runs several corpus searches, reads the results and writes an answer. It routinely takes tens of seconds — longer for complex questions and for questions with attached documents. Corpus agents are faster, since they do one search and distil it.

Set client and proxy read timeouts generously, and prefer `/agents/research/stream` for anything a person is waiting on: the first progress event arrives within a second or two, so the wait is visible rather than blank.

If you put a proxy in front of the streaming endpoint, disable response buffering on that route. The service already sends `X-Accel-Buffering: no`, but not every proxy honours it.

## Where to go next

- [Research agent](02-research-agent.md) — request shape, attachments, conversation history, the JSON response
- [Streaming protocol](03-streaming-protocol.md) — every SSE event, in order, with a worked parser
- [Corpus agents](04-corpus-agents.md) — the five single-corpus search endpoints
- [Schemas](05-schemas.md) — shared objects, and the legal area codes
- [Authentication](../authentication.md) — obtaining and using an access token
