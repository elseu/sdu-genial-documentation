# Corpus Agents

A corpus agent searches one corpus and hands back what it found, distilled. There is no planning, no synthesis and no prose — you get the documents, the findings drawn from them, and the passages those findings came from.

Use these when you are building your own experience on top of the corpus: a search box, a research panel, an enrichment step in your own pipeline. Use the [research agent](02-research-agent.md) when you want a written answer.

---

## Table of contents

1. [The endpoints](#the-endpoints)
2. [Request body](#request-body)
3. [Restricting by legal area](#restricting-by-legal-area)
4. [Example request](#example-request)
5. [Response](#response)
6. [Reading the results](#reading-the-results)
7. [Errors](#errors)

---

## The endpoints

| Endpoint                               | Corpus                                                                                                                                                     |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /agents/search/legislation`      | Dutch national legislation: acts, orders in council, ministerial and policy rules. Searches law currently in force.                                        |
| `POST /agents/search/eu_legislation`   | EU legislation as it applies in the Netherlands: regulations, directives and other EU instruments. Searches instruments currently in force.                |
| `POST /agents/search/case_law`         | Court decisions, from the Hoge Raad and the highest appeal bodies down through the courts of appeal, district courts, supervisory and disciplinary bodies. |
| `POST /agents/search/commentary`       | Scholarly commentary and annotations on legislation and case law.                                                                                          |
| `POST /agents/search/journal_articles` | Articles in Dutch legal and tax journals, discussing legislation and case law.                                                                             |
| `POST /agents/search/practice_notes`   | Practice notes: practical, how-to guidance.                                                                                                                |
| `POST /agents/search/other_sources`    | Everything else: books, blogs, in-depth pieces, news, parliamentary papers, official publications, explanatory memoranda, overviews and tools.             |

They all take the same request and return the same response shape. Only the corpus differs.

## Request body

```json
{
  "legalIssue": "opzegtermijn huurovereenkomst voor bepaalde tijd",
  "legalAreaFacet": ["vast", "cr"]
}
```

| Field            | Type       | Required | Description                                                                  |
| ---------------- | ---------- | -------- | ---------------------------------------------------------------------------- |
| `legalIssue`     | `string`   | Yes      | The legal issue to search for, phrased in Dutch.                             |
| `legalAreaFacet` | `string[]` | No       | Legal area codes to restrict the search to. Omit to search unrestricted.     |

Both `legalIssue` / `legal_issue` and `legalAreaFacet` / `legal_area_facet` are accepted.

**Write `legalIssue` as a legal issue, not as a question.** These endpoints do semantic search over legal text, so they respond best to the way the material is written. `"opzegtermijn huurovereenkomst voor bepaalde tijd"` retrieves well; `"Hoi, ik vroeg me af hoe lang de opzegtermijn is?"` retrieves noticeably worse. Dutch throughout — the corpus is Dutch.

## Restricting by legal area

`legalAreaFacet` takes the short codes listed in [Legal areas](05-schemas.md#legal-areas) — `"arb"` for arbeidsrecht, `"br"` for belastingrecht, and so on.

- **Omitting the field searches every area.** That is the default and usually the right one.
- **An empty array also searches every area.** It does not mean "search nothing".
- Order does not matter and duplicates are collapsed.
- An unknown code is not an error; it simply matches nothing, which can silently narrow a search to zero results. Take the codes from the table rather than typing them by hand.

Restrict when you already know the area from context — a tax product, an employment law workflow. Do not restrict when you are unsure: a question that straddles two areas gets worse results from a narrow facet than from none.

## Example request

```bash
curl -X POST https://genial-api.sdu.nl/v10/agents/search/case_law \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-API-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "legalIssue": "opzegtermijn huurovereenkomst voor bepaalde tijd",
    "legalAreaFacet": ["vast"]
  }'
```

## Response

`200 OK`:

```json
{
  "findings": [
    {
      "document_id": "ECLI_NL_HR_2021_1234",
      "title": "Hoge Raad, 03-09-2021, ECLI:NL:HR:2021:1234",
      "document_date": "2021-09-03",
      "author": null,
      "findings": [
        {
          "finding": "Een huurovereenkomst voor bepaalde tijd van twee jaar of korter eindigt van rechtswege.",
          "relevance": "critical",
          "passage_id": "p-ECLI_NL_HR_2021_1234-12"
        },
        {
          "finding": "De verhuurder moet de huurder wel tijdig informeren over het einde van de huur.",
          "relevance": "relevant",
          "passage_id": "p-ECLI_NL_HR_2021_1234-14"
        }
      ]
    }
  ],
  "references": {
    "ECLI_NL_HR_2021_1234": {
      "document_id": "ECLI_NL_HR_2021_1234",
      "display_title": "Hoge Raad, 03-09-2021, ECLI:NL:HR:2021:1234",
      "original_source_url": "https://…",
      "passages": [
        { "identifier": "p-ECLI_NL_HR_2021_1234-12", "text": "…" },
        { "identifier": "p-ECLI_NL_HR_2021_1234-14", "text": "…" }
      ],
      "published": [],
      "about": []
    }
  },
  "errors": []
}
```

| Field        | Type                                              | Description                                                                     |
| ------------ | ------------------------------------------------- | ------------------------------------------------------------------------------- |
| `findings`   | [`DistilledSource[]`](05-schemas.md#distilledsource) | One entry per document found, with the findings drawn from it.                   |
| `references` | `object`                                          | Map of `document_id` → [`SearchResult`](05-schemas.md#searchresult). The documents themselves. |
| `errors`     | [`Error[]`](01-overview.md#errors)                   | Non-fatal problems. Empty on a clean search.                                     |

Responses are `snake_case`, including the nested objects. Note the doubled name: `findings[].findings` is the list of findings inside one document.

## Reading the results

`findings` and `references` are two views of the same search. `findings` is what the agent concluded; `references` is the source material it concluded it from. They are joined on `document_id`.

To render a result:

1. For each entry in `findings`, look up `references[entry.document_id]` for the title, URL and publication metadata.
2. For each finding inside it, look up the passage in that reference whose `identifier` equals `finding.passage_id`. That passage is the text the finding rests on — quote it, or link to it.
3. Sort or filter on `relevance`: `critical`, `relevant` or `marginal`, in that order of importance.

Every finding carries a `passage_id`, so every claim in the response is traceable to a specific piece of text. Show that provenance. Legal users check.

**An empty `findings` array is a normal outcome.** It means nothing in that corpus bore on the issue — a case law search on a purely statutory question, for example. Try a different corpus or a broader `legalIssue` before treating it as a failure.

## Errors

Request-level failures (`401`, `403`, `422`, `429`) are described in [Errors](01-overview.md#errors).

A `200` with entries in `errors` means the search ran but something went wrong along the way — most often `No search results found for the user's query.`, which pairs with an empty `findings`. Show the Dutch `user_message`; log the rest.
