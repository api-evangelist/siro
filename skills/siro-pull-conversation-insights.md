---
name: Pull recording summaries and AI entity extractions
description: Retrieve a Siro recording's conversation summary and AI-extracted fields (budget, objections, decision-makers, timelines) to push back into your system of record.
api: openapi/siro-external-api-openapi.json
operations:
  - "POST /v1/core/oauth/apps"
  - "POST /v1/core/oauth/apps/{clientId}/access-token"
  - "GET /v1/core/recordings/{id}"
  - "GET /v1/core/recordings/{id}/summaries"
---

# Pull recording summaries and AI entity extractions

Use this skill to fetch the coaching/conversation intelligence Siro produces for a recording and
sync it back into your CRM or data warehouse.

## Auth
- Recording detail and entity-extraction reads are **user-scoped** and use the OAuth access token,
  sent as `x-siro-auth-token: <access-token>` (NOT the org Bearer token).
- Base URL for these reads: `https://api.siro.ai`.

## Steps
1. **Create an OAuth app** (once) — `POST /v1/core/oauth/apps` to obtain a `client_id`/`client_secret`.
2. **Mint an access token** — `POST /v1/core/oauth/apps/{clientId}/access-token` at user scope.
3. **Read the recording** — `GET /v1/core/recordings/{id}?showSummary=true&showEntityExtractions=true`
   to get the recording plus its LLM summary and extracted fields with CRM field mappings.
4. **(Optional) Summaries only** — `GET /v1/core/recordings/{id}/summaries` when you just need summaries.

## Rules (from conventions/ + errors/)
- Use `showSummary` / `showEntityExtractions` query flags (default false) to include enriched data.
- React to the `integrations.recordingProcessed` webhook to know when summary + extractions are ready,
  rather than polling; verify the Svix `svix-*` signature headers on the raw body.
- List endpoints paginate by cursor or page (`cursor`/`page`/`pageSize`); cursor wins if both are sent.
- Handle 401/403 (token scope), 404 (unknown recording) per errors/siro-problem-types.yml.
