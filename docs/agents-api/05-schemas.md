# Schemas

The objects that appear in more than one place, and the legal area codes.

Every schema on this page is `snake_case`, matching the responses. Request envelopes additionally accept `camelCase` — see [Naming conventions](01-overview.md#naming-conventions).

---

## Table of contents

1. [`SearchResult`](#searchresult)
2. [`Passage`](#passage)
3. [`DistilledSource`](#distilledsource)
4. [`DistilledFinding`](#distilledfinding)
5. [`ExecutionPlan`](#executionplan)
6. [`Error`](#error)
7. [Legal areas](#legal-areas)
8. [Answer templates](#answer-templates)

---

## `SearchResult`

A document from the corpus, with the passages that matched. Appears as the value type of every `references` map and as the `data` of a `data-reference` event.

| Field                 | Type                              | Description                                                                    |
| --------------------- | --------------------------------- | ------------------------------------------------------------------------------ |
| `document_id`         | `string`                          | Stable corpus identifier. The key this document is stored under.               |
| `display_title`       | `string \| null`                  | Title written for display. Prefer this.                                        |
| `title`               | `string \| null`                  | Raw title. Fall back to this.                                                  |
| `original_source_url` | `string \| null`                  | Where to link the user.                                                        |
| `snippet`             | `string \| null`                  | Short extract.                                                                 |
| `full_text`           | `string \| null`                  | Whole document text, when it was retrieved in full.                            |
| `passages`            | [`Passage[]`](#passage) `\| null` | The passages that matched. Citations resolve into this array.                  |
| `is_sdu_commentary`   | `boolean \| null`                 | Whether the document is Sdu commentary.                                        |
| `is_repealed`         | `boolean \| null`                 | Whether the regulation has been repealed.                                      |
| `ro_source`           | `string \| null`                  | Source system the document came from.                                          |
| `publisher`           | `string \| null`                  | Publisher.                                                                     |
| `published`           | `Publication[]`                   | Publication records. See below.                                                |
| `creator`             | `object \| null`                  | `{ label, value }` — the author, as the source records them.                   |
| `about`               | `AboutEntry[]`                    | What the document is about. Carries case metadata for case law. See below.     |
| `pma_fields`          | `object \| null`                  | `{ identifies, references, about }`, each a `string[]` of related identifiers. |
| `article_numbers`     | `string[] \| null`                | Article numbers covered, for legislation.                                      |
| `document_date`       | `string \| null`                  | Document date.                                                                 |

Only `document_id` is guaranteed present. Treat every other field as optional and code the fallbacks — the corpus spans very different document types and few fields are meaningful for all of them.

### `Publication` (inside `published`)

| Field              | Type             | Description                                 |
| ------------------ | ---------------- | ------------------------------------------- |
| `concise_citation` | `string \| null` | The short citation form, e.g. `NJ 2021/45`. |
| `publication_date` | `string \| null` | Date of publication.                        |
| `publication_name` | `object \| null` | `{ title, identifier }` of the publication. |

### `AboutEntry` (inside `about`)

| Field            | Type             | Description                                                      |
| ---------------- | ---------------- | ---------------------------------------------------------------- |
| `case_date`      | `string[]`       | Decision dates, for case law.                                    |
| `case_number`    | `string[]`       | Case numbers.                                                    |
| `alt_identifier` | `string[]`       | Alternative identifiers, e.g. an ECLI.                           |
| `valid`          | `object \| null` | `{ start, end, status }` — the validity window, for legislation. |

## `Passage`

```json
{ "identifier": "p-BWBR0005290-7-2", "text": "De huurovereenkomst eindigt…" }
```

| Field        | Type     | Description                                       |
| ------------ | -------- | ------------------------------------------------- |
| `identifier` | `string` | Passage id. This is what a citation marker names. |
| `text`       | `string` | The passage text, verbatim from the document.     |

Passage identifiers are what the whole citation mechanism turns on: an answer cites `identifier`, and you resolve it back to the containing `SearchResult`. Never assume a passage identifier is derivable from a document id.

## `DistilledSource`

One document, reduced to what it contributes to the question. Appears in `findings` on the corpus endpoints, in the `*_results` arrays of the research response, and in search progress events.

| Field           | Type                                      | Description                              |
| --------------- | ----------------------------------------- | ---------------------------------------- |
| `document_id`   | `string`                                  | Joins to `references[document_id]`.      |
| `findings`      | [`DistilledFinding[]`](#distilledfinding) | What this document says about the issue. |
| `title`         | `string \| null`                          | Document title.                          |
| `document_date` | `string \| null`                          | Document date.                           |
| `author`        | `string \| null`                          | Author, where known.                     |

## `DistilledFinding`

| Field        | Type     | Description                                                                   |
| ------------ | -------- | ----------------------------------------------------------------------------- |
| `finding`    | `string` | One point the document makes about the issue, in Dutch.                       |
| `relevance`  | `string` | `critical`, `relevant` or `marginal`.                                         |
| `passage_id` | `string` | The passage this finding came from. Resolves into that document's `passages`. |

`relevance` is the agent's judgement of how much the finding bears on the issue, not a search score. `critical` means the answer turns on it.

## `ExecutionPlan`

One search the research agent decided to run. Appears in `researched_plans` and in the `data-planner` progress event.

| Field              | Type               | Description                                                                                                                     |
| ------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| `legal_issue`      | `string`           | The Dutch query this search ran.                                                                                                |
| `agent`            | `string`           | `legislation`, `eu_legislation`, `case_law`, `commentary`, `journal_articles`, `practice_notes`, `other_sources` or `document`. |
| `legal_area_facet` | `string[] \| null` | Legal area codes the search was restricted to. `null` means unrestricted.                                                       |

## `Error`

| Field          | Type             | Description                        |
| -------------- | ---------------- | ---------------------------------- |
| `message`      | `string`         | English, for logs.                 |
| `user_message` | `string`         | Dutch, safe to display as-is.      |
| `details`      | `object \| null` | Optional string-to-string context. |

See [Errors](01-overview.md#errors) for the full list and when each occurs.

## Legal areas

Two spellings of the same concept appear in the API, and they are not interchangeable:

- **Codes** — short strings like `arb`. Used in `legalAreaFacet` on the corpus endpoints and in `legal_area_facet` inside an `ExecutionPlan`.
- **Labels** — Dutch names like `arbeidsrecht`. Used in `legal_areas` on the research response and in the `data-planner` event.

| Code   | Label                                |
| ------ | ------------------------------------ |
| `aan`  | aanbestedingsrecht                   |
| `arb`  | arbeidsrecht                         |
| `br`   | belastingrecht                       |
| `bpr`  | burgerlijk procesrecht               |
| `cr`   | civiel recht                         |
| `eu`   | europees recht                       |
| `fin`  | financieel recht                     |
| `gzr`  | gezondheidsrecht                     |
| `hr`   | handelsrecht                         |
| `inf`  | ict, telecommunicatie- en mediarecht |
| `ins`  | insolventierecht                     |
| `ie`   | intellectueel eigendomsrecht         |
| `int`  | internationaal recht                 |
| `jgd`  | jeugdrecht                           |
| `omg`  | omgevingsrecht                       |
| `or`   | ondernemingsrecht                    |
| `ovr`  | overig                               |
| `pfr`  | personen- en familierecht            |
| `alg`  | recht algemeen                       |
| `soz`  | socialezekerheidsrecht               |
| `sb`   | staats- en bestuursrecht             |
| `sr`   | strafrecht                           |
| `vast` | vastgoed- en huurrecht               |
| `vrm`  | vermogensrecht                       |

## Answer templates

The shape the research agent gives its answer, reported in the [`data-template-selector`](03-streaming-protocol.md#progress-events) event. It is chosen from the question; you cannot set it.

| Value          | Answer shape                                                                 |
| -------------- | ---------------------------------------------------------------------------- |
| `cirac`        | Structured legal analysis: conclusion, issue, rule, application, conclusion. |
| `memo`         | A legal memo.                                                                |
| `email`        | An email, drafted for sending.                                               |
| `list_sources` | A list of sources rather than an argument.                                   |
| `simple`       | A direct answer to a simple question.                                        |
| `transform`    | A rewrite or transformation of material supplied in the question.            |
