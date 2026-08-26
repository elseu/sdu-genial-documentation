This repository contains the documentation for connecting to the Sdu GenIA-L API. Below, you'll find links to the various documentation files stored in the `/docs` directory.

## Documentation

Start with authentication, then the overview — it links on to everything else.

- [Authentication](./docs/authentication.md) — obtaining an access token, tenants and access levels

### [Agents API](./docs/agents-api/) — v10 and later

1. [Overview](./docs/agents-api/01-overview.md) — base URL, endpoints, headers, rate limits and errors
2. [Research agent](./docs/agents-api/02-research-agent.md) — `POST /agents/research` and `POST /agents/research/stream`
3. [Streaming protocol](./docs/agents-api/03-streaming-protocol.md) — every SSE event, in order, with a worked parser
4. [Corpus agents](./docs/agents-api/04-corpus-agents.md) — the five `POST /agents/search/*` endpoints
5. [Schemas](./docs/agents-api/05-schemas.md) — shared objects and the legal area codes

## Changelog

### V10

#### 10.0.0

- Added the Agents API: a research agent (`/agents/research`, `/agents/research/stream`) and five corpus search agents (`/agents/search/*`). See the [Agents API Overview](./docs/agents-api/01-overview.md).
- The research stream is emitted in the AI SDK UI Message Stream format, replacing the content-part stream of `/message`. See [Streaming protocol](./docs/agents-api/03-streaming-protocol.md).
- Answers cite passages rather than documents; every citation resolves to the exact passage it rests on. See [Citations](./docs/agents-api/02-research-agent.md#citations).
- Removed the Response Parsing and Prompt Enhancer documentation, which covered the `/message` and `/enhance-prompt` endpoints of earlier versions.

### V5

#### 5.0.0 (11-05-2026)

- BREAKING CHANGE: All endpoints require `X-Auth-Tenant-Id` header (see [Authentication](./docs/authentication.md)).
- `chunk_url` deprecated in favor of `original_source_url`.
- Rework citation manager for extra citation validation.
- Richer reasoning steps.
- Improvements to synthesis constraints for more output control.

### V4

#### 4.0.0 (20-10-2025)

- BREAKING CHANGE: Introduced Reasoning of the agent as output into the response
- BREAKING CHANGE: Stopped sending contentpart type <SYSTEM> messages and replaced them with a new format for Reasoning
- Improvements to markdown formatting in answers
- Speed improvements by moving some core components back to GPT 4.1
- Add the new Prompt Enhancer feature through the `/enhance-prompt` endpoint allowing for realtime (or workflow integrated) prompt suggestions.

### V3

#### 3.6.0 (09-09-2025)

- Increase max input query size to 6000 characters
- Add keep alive (zero-width-byte) chunks for long idle time queries

#### 3.3.0 (21-08-2025)

- Migrate GPT models from GPT-4o to GPT-4.1 and GPT-5
- Introduction of agentic workflows for composite (long/complex) questions
- Added routes for `/docs` and `/openapi.json` (both need the accesstoken in Authorization headers)
  - https://genial-api.sdu.nl/v3/openapi.json
  - https://genial-api.sdu.nl/v3/docs
- Deprecate `applicationKey` from request parameters

### V2

#### 2.0.0 (01-07-2025)

- Change endpoint from `/step` to `/message`
- Added `smart_actions` to the `metadata` object that can be used as followup prompts in context to the current conversation.
