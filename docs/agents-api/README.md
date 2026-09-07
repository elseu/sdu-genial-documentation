# GenIA-L Agents API

Documentation for the agent endpoints of the GenIA-L API, available from **v10** onwards at
`https://genial-api.sdu.nl/{VERSION}/agents/...`.

Read them in order — each page assumes the one before it.

| #   | Page                                           | Covers                                            |
| --- | ---------------------------------------------- | ------------------------------------------------- |
| 1   | [Overview](01-overview.md)                     | Base URL, endpoints, headers, rate limits, errors |
| 2   | [Research agent](02-research-agent.md)         | `/agents/research` and `/agents/research/stream`  |
| 3   | [Streaming protocol](03-streaming-protocol.md) | Every SSE event, in order, with a worked parser   |
| 4   | [Corpus agents](04-corpus-agents.md)           | The `/agents/search/*` endpoints                  |
| 5   | [Schemas](05-schemas.md)                       | Shared objects and the legal area codes           |

Authentication is documented separately, in [Authentication](../authentication.md). Start there:
every endpoint on these pages needs an access token and a tenant id.
