# The Research Agent

The research agent takes a legal question and answers it in prose, with every claim cited back to a passage of a real document. Internally it classifies the question, picks the legal areas it falls under, plans a set of searches, runs them across the corpora in parallel, reads what comes back, checks whether it has enough, and only then writes.

You never drive those steps. You send one question and take one answer.

---

## Table of contents

1. [Choosing streaming or JSON](#choosing-streaming-or-json)
2. [Request body](#request-body)
   - [Message parts](#message-parts)
   - [Conversation history](#conversation-history)
3. [Example requests](#example-requests)
4. [The JSON response](#the-json-response)
5. [Citations](#citations)
6. [Follow-up questions](#follow-up-questions)
7. [Errors](#errors)

---

## Choosing streaming or JSON

|                  | `POST /agents/research/stream`             | `POST /agents/research`       |
| ---------------- | ------------------------------------------ | ----------------------------- |
| Answers with     | `text/event-stream`                        | `application/json`            |
| First byte       | within a second or two                     | after the full turn completes |
| Answer text      | streamed in deltas                         | one `content` string          |
| Citation markers | `<sup>passage_id</sup>`                    | `【†passage_id†】`            |
| Cited documents  | `data-reference` events, as they are cited | the `references` map          |
| Progress visible | yes, step by step                          | no                            |

Both run exactly the same agent and accept exactly the same request body. Use the streaming endpoint whenever a person is waiting for the answer; use the JSON endpoint for batch jobs, evaluation runs and anything that stores the answer rather than showing it live.

The rest of this page covers the request body, which is shared, and the JSON response. The stream is documented in [Streaming protocol](03-streaming-protocol.md).

## Request body

```json
{
  "message": {
    "role": "user",
    "parts": [
      {
        "type": "text",
        "text": "Welke opzegtermijn geldt bij een huurovereenkomst voor bepaalde tijd?"
      }
    ]
  }
}
```

| Field               | Type       | Required | Description                                                   |
| ------------------- | ---------- | -------- | ------------------------------------------------------------- |
| `message`           | `object`   | Yes      | The turn to answer.                                           |
| `message.parts`     | `array`    | Yes      | The content of the turn. See [Message parts](#message-parts). |
| `message.role`      | `string`   | No       | Only `"user"` is accepted. Defaults to `"user"`.              |
| `message.id`        | `uuid`     | No       | Your id for this message. Not used to answer it.              |
| `message.createdAt` | `datetime` | No       | ISO 8601. Not used to answer it.                              |

`message.parts` is the only field you have to send. A question with no history, no attachments and no ids is a complete, valid request.

### Message parts

Parts follow the [AI SDK](https://ai-sdk.dev) `UIMessage` part format. Three types matter in a user turn:

#### `text`

The question itself.

```json
{
  "type": "text",
  "text": "Mag een werkgever een concurrentiebeding eenzijdig wijzigen?"
}
```

Send one text part per turn. If you send several, they are read in order as one question.

Other AI SDK part types (`reasoning`, `file`, `step-start`, `tool-*`, `data-*`) are accepted by the schema so you can replay a stored conversation unchanged, but they contribute nothing to a user turn.

### Conversation history

The agent is stateless per request. It sees history only if you ask it to load some. The support for self managed history is to be added.

## Example requests

### Streaming

```bash
curl -N -X POST https://genial-api.sdu.nl/v10/agents/research/stream \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-API-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "parts": [
        { "type": "text", "text": "Welke opzegtermijn geldt bij een huurovereenkomst voor bepaalde tijd?" }
      ]
    }
  }'
```

`-N` disables curl's own buffering, so you see events as they arrive.

### JSON

```bash
curl -X POST https://genial-api.sdu.nl/v10/agents/research \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-API-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "parts": [
        { "type": "text", "text": "Welke opzegtermijn geldt bij een huurovereenkomst voor bepaalde tijd?" }
      ]
    }
  }'
```

## The JSON response

`POST /agents/research` returns `200` with:

```json
{
  "content": "Bij een huurovereenkomst voor bepaalde tijd geldt…【†p-BWBR0005290-7-2†】",
  "references": {
    "BWBR0005290": {
      "document_id": "BWBR0005290",
      "display_title": "Burgerlijk Wetboek Boek 7",
      "original_source_url": "https://…",
      "passages": [
        {
          "identifier": "p-BWBR0005290-7-2",
          "text": "De huurovereenkomst eindigt…"
        }
      ],
      "published": [],
      "about": []
    }
  },
  "legislation_results": [
    {
      "document_id": "BWBR0005290",
      "title": "Burgerlijk Wetboek Boek 7",
      "document_date": "2024-01-01",
      "author": null,
      "findings": [
        {
          "finding": "Een huurovereenkomst voor bepaalde tijd eindigt van rechtswege…",
          "relevance": "critical",
          "passage_id": "p-BWBR0005290-7-2"
        }
      ]
    }
  ],
  "eu_legislation_results": [],
  "case_law_results": [],
  "commentary_results": [],
  "journal_articles_results": [],
  "practice_notes_results": [],
  "other_sources_results": [],
  "followup_queries": ["Wat gebeurt er bij stilzwijgende verlenging?"],
  "researched_plans": [
    {
      "legal_issue": "opzegtermijn huurovereenkomst bepaalde tijd",
      "agent": "legislation",
      "legal_area_facet": ["vast"],
      "retrieval_mode": null
    }
  ],
  "research_gap": "",
  "replan_count": 0,
  "legal_areas": ["vastgoed- en huurrecht"],
  "errors": []
}
```

| Field                      | Type                                                 | Description                                                                                                                |
| -------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `content`                  | `string`                                             | The answer, in Dutch Markdown, with `【†passage_id†】` citation markers inline.                                              |
| `references`               | `object`                                             | Map of `document_id` → [`SearchResult`](05-schemas.md#searchresult). Everything the answer cites.                          |
| `legislation_results`      | [`DistilledSource[]`](05-schemas.md#distilledsource) | What the Dutch legislation search found and what was distilled from it.                                                    |
| `eu_legislation_results`   | `DistilledSource[]`                                  | Same, for EU regulations, directives and other EU instruments.                                                             |
| `case_law_results`         | `DistilledSource[]`                                  | Same, for court decisions.                                                                                                 |
| `commentary_results`       | `DistilledSource[]`                                  | Same, for scholarly commentary and annotations.                                                                            |
| `journal_articles_results` | `DistilledSource[]`                                  | Same, for articles in legal and tax journals.                                                                              |
| `practice_notes_results`   | `DistilledSource[]`                                  | Same, for practical guidance.                                                                                              |
| `other_sources_results`    | `DistilledSource[]`                                  | Same, for books, blogs, news, official publications and other secondary material.                                          |
| `followup_queries`         | `string[]`                                           | Suggested next questions. See [Follow-up questions](#follow-up-questions).                                                 |
| `researched_plans`         | [`ExecutionPlan[]`](05-schemas.md#executionplan)     | The searches that were actually run, in order.                                                                             |
| `research_gap`             | `string`                                             | What the verification step judged to be missing. Empty when nothing was.                                                   |
| `replan_count`             | `integer`                                            | How many times the agent went back and searched again. `0` means it got there first time.                                  |
| `legal_areas`              | `string[]`                                           | The Dutch legal area labels the question was classified under, one to three. See [Legal areas](05-schemas.md#legal-areas). |
| `errors`                   | [`Error[]`](01-overview.md#errors)                   | Non-fatal problems. Empty on a clean turn — **always check it**.                                                           |

Note that the `*_results` arrays hold everything that was **found**, which is a superset of what the answer ended up **citing**. `references` holds what was cited. Render from `references`; use the `*_results` arrays when you want to show the full research trail.

## Citations

Citations point at a **passage**, not a document. A single document can be cited several times through different passages.

In the JSON response, the answer carries markers in the form:

```plaintext
【†passage_id†】
```

To resolve one:

1. Walk `references`; each entry has a `passages` array.
2. Find the passage whose `identifier` equals the `passage_id` in the marker.
3. The entry containing it is the cited document; `passage.text` is the exact text the claim rests on.

A rendering pass usually replaces each marker with a footnote number, numbering by first appearance, and groups the footnotes by document.

> The streaming endpoint does this resolution for you: it emits `<sup>passage_id</sup>` in the text and a `data-reference` event carrying the document. See [Streaming protocol](03-streaming-protocol.md#citations).

Markers use the full-width brackets `【` `】` with a dagger `†` immediately inside each. Match on the exact characters. Very occasionally a malformed marker survives into `content`; drop anything that does not resolve rather than showing it.

## Follow-up questions

`followup_queries` holds suggested next questions in Dutch, derived from what the research turned up. They are meant to be offered to the user as one-click prompts. The array can be empty.

## Errors

Request-level failures (`401`, `403`, `422`, `429`) are described in [Errors](01-overview.md#errors).

Everything else surfaces in the `errors` array with `200 OK`. The common cases:

| `message`                                       | What it means                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------- |
| `No search results found for the user's query.` | Nothing relevant was found. `content` will not carry a real answer. |
| `API error`                                     | An upstream dependency failed mid-turn.                             |

Each carries a Dutch `user_message` written to be displayed as-is. Show that, not `message`.
