# Corpus Agents

A corpus agent searches one corpus and hands back what it found, distilled. There is no planning, no synthesis and no prose — you get the documents, the findings drawn from them, and the passages those findings came from.

Use these when you are building your own experience on top of the corpus: a search box, a research panel, an enrichment step in your own pipeline. Use the [research agent](02-research-agent.md) when you want a written answer.

---

## Table of contents

1. [The endpoints](#the-endpoints)
2. [Choosing an agent](#choosing-an-agent)
3. [Request body](#request-body)
4. [Restricting by legal area](#restricting-by-legal-area)
5. [Example request](#example-request)
6. [Response](#response)
7. [Reading the results](#reading-the-results)
8. [Errors](#errors)

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

## Choosing an agent

Most substantive questions need more than one agent. Call them in parallel, one request per agent per legal concept, and merge the findings yourself. Phrase `legalIssue` for the agent you are calling. The styles below differ on purpose, because each corpus retrieves well on a different kind of text.

| Agent              | Call it when you need                                                                   | What it searches                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | How to phrase `legalIssue`                                                                                                                                                                                                                                   |
| ------------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `legislation`      | Findings on a Dutch statute, provision or statutory concept                             | Wetten, AMvB's, ministeriële regelingen, beleidsregels, koninklijke besluiten. Current version only.                                                                                                                                                                                                                                                                                                                                                                                                                                             | A keyword-dense statutory concept of three to five content words, no facts from the case. `verzuim ingebrekestelling nakoming verbintenis`, `opzegging huurovereenkomst woonruimte`. One concept per call.                                                   |
| `eu_legislation`   | An EU regulation or directive that governs the matter, whether or not the user named it | EU regulations and directives as they apply in the Netherlands. Data protection, VAT, consumer rights, competition, product safety, social security coordination and the like.                                                                                                                                                                                                                                                                                                                                                                   | Same keyword style, but name the instrument the way a Dutch lawyer says it. `algemene verordening gegevensbescherming rechtmatigheid verwerking`, `btw-richtlijn plaats van dienst`. Pair it with a `legislation` call for the Dutch implementing provision. |
| `case_law`         | How courts have applied the law to facts like these                                     | Rulings from all Dutch courts and tribunals plus European instances. Three searches run at once: lower and other courts, the Hoge Raad and highest appeal courts, and up to five landmark ECLIs.                                                                                                                                                                                                                                                                                                                                                 | A fluent Dutch sentence that reads like a case summary and includes the concrete circumstances (who, what happened, amounts, durations). Leave out request words such as "uitspraken uit 2024 over".                                                         |
| `commentary`       | The current, maintained statement of a rule, anchored to a provision                    | Scholarly commentary and annotations on legislation and case law, each piece anchored to a specific provision.                                                                                                                                                                                                                                                                                                                                                                                                                                   | A topic phrase around the provision or doctrine.                                                                                                                                                                                                             |
| `journal_articles` | Argument rather than settled statement                                                  | Articles in legal and tax journals.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | A topic phrase around the issue itself, not an article number, and without period words.                                                                                                                                                                     |
| `practice_notes`   | Procedural or how-to guidance                                                           | Practice notes and practical step-by-step material.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | A topic phrase around the task the practitioner has to carry out.                                                                                                                                                                                            |
| `other_sources`    | Background, legislative history, official publications, news or tools                   | Kamerstukken, Kamervragen, Handelingen and the memorie van toelichting, moties, amendementen and rapporten within them. Staatsblad, Staatscourant, Tractatenblad, permits, collective labour agreement notices, ministerial regulations and policy rules. Belastingdienst kennisgroepstandpunten. Legal news and current-awareness bulletins. Books and handbooks. Practitioner blogs. Model documents, checklists, decision trees and calculators. Legislative tracking overviews. Convenanten, gedragscodes and industry terms and conditions. | A topic phrase around the matter, for instance `memorie van toelichting Wet bescherming klokkenluiders`.                                                                                                                                                     |

Combining agents:

- A statutory concept with a dispute or interpretation angle needs `legislation` and `case_law`. Add `commentary` when the reader needs the settled reading of the provision.
- An EU instrument governing the matter needs `eu_legislation` and `legislation` both. A directive only works through its Dutch implementing provision, and the two corpora are separate.
- `commentary` and `journal_articles` answer different questions. Commentary gives the maintained statement of the rule. Journal articles give argument. Call `journal_articles` for a contested or unsettled point, a development too recent for a maintained commentary to have absorbed, a case note on a specific ruling, or a niche or cross-domain question. Call both when the matter has both angles.
- A matter that combines a general doctrine (wanprestatie, onrechtmatige daad) with a specific statutory regime (oplevering bij aanneming, huurrecht) gets one `legislation` call per concept. Combined queries retrieve poorly.
- A purely definitional lookup of a statutory term, with no dispute angle, needs `legislation` alone.

## Request body

```json
{
  "legalIssue": "opzegtermijn huurovereenkomst voor bepaalde tijd",
  "legalAreaFacet": ["vast", "cr"]
}
```

| Field            | Type       | Required | Description                                                              |
| ---------------- | ---------- | -------- | ------------------------------------------------------------------------ |
| `legalIssue`     | `string`   | Yes      | The legal issue to search for, phrased in Dutch.                         |
| `legalAreaFacet` | `string[]` | No       | Legal area codes to restrict the search to. Omit to search unrestricted. |

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

| Field        | Type                                                 | Description                                                                                    |
| ------------ | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `findings`   | [`DistilledSource[]`](05-schemas.md#distilledsource) | One entry per document found, with the findings drawn from it.                                 |
| `references` | `object`                                             | Map of `document_id` → [`SearchResult`](05-schemas.md#searchresult). The documents themselves. |
| `errors`     | [`Error[]`](01-overview.md#errors)                   | Non-fatal problems. Empty on a clean search.                                                   |

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
