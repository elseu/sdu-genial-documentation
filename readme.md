This repository contains the documentation for connecting to the Sdu GenIA-L API. Below, you'll find links to the various documentation files stored in the `/docs` directory.

## Documentation

- [Authentication](./docs/authentication.md)
- [Response Parsing](./docs/response_parsing.md)

## Changelog

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
