---
name: Sync CRM engagements and link Siro recordings
description: Push appointments and deals from your CRM into Siro so field-sales recordings are automatically matched to the right engagement and opportunity.
api: openapi/siro-external-api-openapi.json
operations:
  - "PUT /v1/integrations/sync/engagements"
  - "PUT /v1/integrations/sync/opportunities"
  - "GET /v1/integrations/engagements/{id}"
---

# Sync CRM engagements and link Siro recordings

Use this skill to keep Siro's engagement/opportunity graph in step with your CRM so that
each recorded sales conversation links to the correct appointment and deal.

## Auth
- Send `Authorization: Bearer <organization-api-token>` (generate in the Siro admin: Person icon -> API Tokens).
- Base URL: `https://functions.siro.ai/api-externalApi`.

## Steps
1. **Sync the appointment** — `PUT /v1/integrations/sync/engagements` with the meeting/appointment,
   including your CRM-native `externalId` and `integrationConnectionId`, plus the `account` context and an
   `activityType` (MEETING, APPOINTMENT or EVENT show in the appointment list).
2. **Sync the deal** — `PUT /v1/integrations/sync/opportunities` with `amount`, `disposition`, customer
   name and the same `externalId` + `integrationConnectionId` reference so recordings can attach to it.
3. **Confirm linkage** — `GET /v1/integrations/engagements/{id}` to read the engagement and its linked
   recording once the rep has recorded.

## Rules (from conventions/ + errors/)
- Target resources either by Siro `id` or by `externalId` together with `integrationConnectionId`.
- Sync operations are `PUT` upserts and are idempotent on the resource key; re-sending is safe.
- Stay under 1000 requests per API token per minute; back off on errors.
- Handle 400/422 (validation), 401/403 (token scope), 404 (unknown reference) per errors/siro-problem-types.yml.
- Prefer a webhook subscription (`integrations.recordingLinked`) over polling to learn when a recording links.
