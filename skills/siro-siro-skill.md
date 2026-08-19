---
name: Siro
description: Use when building CRM integrations, syncing sales conversations and appointment data, retrieving AI-generated insights from recordings, setting up webhooks for real-time event processing, or uploading phone recordings to Siro for analysis and coaching.
metadata:
    mintlify-proj: siro
    version: "1.0"
---

# Siro Skill Reference

## Product summary

Siro is a conversation intelligence platform that records, transcribes, and analyzes sales conversations. Agents use Siro to build bidirectional CRM integrations that sync appointments and opportunities to Siro, automatically link recordings to CRM records, extract conversation insights, and push enriched data back to the CRM. The platform exposes two API base URLs: `https://functions.siro.ai/api-externalApi/v1` for org-wide operations (syncing data, managing tokens, webhooks) and `https://api.siro.ai/v1` for user-scoped operations (recording details, extractions, scorecard metrics). Authentication uses organization API tokens (Bearer) or OAuth access tokens (x-siro-auth-token header). Rate limit: 1000 requests per token per minute. Primary docs: https://docs.siro.ai

## When to use

Reach for this skill when:

- **Building a custom CRM integration** — syncing appointments (engagements), opportunities, and accounts from your system to Siro so recordings auto-link to CRM context
- **Uploading phone recordings** — ingesting call recordings from a VoIP system into Siro for transcription, analysis, and CRM linking
- **Retrieving conversation insights** — pulling summaries, entity extractions, and scorecard metrics from processed recordings to write back to your CRM
- **Setting up webhooks** — subscribing to `integrations.recordingProcessed` and `integrations.recordingLinked` events to trigger downstream workflows
- **Triggering recordings from your app** — using deep links to start/stop Siro recordings without leaving your mobile app
- **Managing OAuth apps and tokens** — creating OAuth apps for user-scoped API access (required for recording details and extractions)

## Quick reference

### API Base URLs

| Purpose | Base URL | Auth Header |
|---------|----------|-------------|
| Org-wide (sync, tokens, webhooks) | `https://functions.siro.ai/api-externalApi/v1` | `Authorization: Bearer <org-api-token>` |
| User-scoped (recordings, extractions, scorecards) | `https://api.siro.ai/v1` | `x-siro-auth-token: <oauth-access-token>` |

### Key Endpoints

| Task | Method | Endpoint | Auth |
|------|--------|----------|------|
| Get org API token | Manual | Siro dashboard → Person Icon → API Tokens | N/A |
| Sync engagement (appointment) | PUT | `/integrations/sync/engagements` | Org token |
| Sync opportunity (deal) | PUT | `/integrations/sync/opportunities` | Org token |
| Get engagement details | GET | `/integrations/engagements/{id}` | Org token |
| Create OAuth app | POST | `/core/oauth/apps` | Org token |
| Create OAuth access token | POST | `/core/oauth/apps/{clientId}/access-token` | Org token |
| Get recording details + summary | GET | `/core/recordings/{recordingId}?showSummary=true&showEntityExtractions=true` | OAuth token |
| Get entity extractions | GET | `/core/entities/extractions/{recordingId}` | OAuth token |
| Get scorecard metrics | GET | `/core/scorecards/scorecard-metrics/values` | OAuth token |
| Upload phone recording (signed URL) | POST | `/core/recordings/signed-urls` | OAuth token |
| Upload phone recording (create) | POST | `/core/recordings/upload` | OAuth token |
| Subscribe to webhooks | Manual | Siro dashboard → Organization Admin → Webhooks | N/A |

### Data Model Mapping

| Your System | Siro Concept | API Field | Notes |
|-------------|--------------|-----------|-------|
| Sales rep / roster | External User | `User` with `externalId`, `email`, `name` | Maps to Siro User for workspace login |
| Customer | Account | `Account` with `externalId` | Synced via engagement/opportunity payloads |
| Deal / opportunity | Opportunity | `Opportunity` with `externalId`, `amount`, `disposition` | Adds deal context to recordings |
| Appointment / meeting | Engagement | `Engagement` with `externalId`, `startTime`, `endTime` | Core link: Recording ↔ Engagement ↔ Opportunity |
| Recording | Recording | `recordingId` | Siro-internal UUID; linked via webhooks |

### Webhook Events

| Event | Fires When | Payload Includes |
|-------|-----------|------------------|
| `integrations.recordingProcessed` | Siro finishes transcript, summary, extractions | `recordingProcessed: true`, CRM context |
| `integrations.recordingLinked` | Recording linked to CRM record | `recordingLinked: true`, CRM context |

**Critical:** Both events report current state for both flags. Act only when both `recordingProcessed` and `recordingLinked` are `true`.

### Deep Link Format

```
https://app.siro.ai/record?action={start|stop|restart}&title={title}&conversationType={uuid}&redirectUrl={encoded-url}&integrationConnectionId={id}&crmObjects={encoded-array}
```

## Decision guidance

### When to use org API token vs OAuth access token

| Scenario | Use | Why |
|----------|-----|-----|
| Syncing engagements, opportunities, accounts | Org API token | Org-wide data sync; no user scope needed |
| Creating OAuth apps and tokens | Org API token | Admin operation; requires org-level access |
| Fetching recording details, summaries, extractions | OAuth access token | User-scoped; tied to a specific Siro User |
| Polling engagement status | Org API token | Org-wide read; no user scope needed |
| Retrieving scorecard metrics | OAuth access token | User-scoped; tied to a specific Siro User |

### When to use webhooks vs polling

| Approach | Use When | Pros | Cons |
|----------|----------|------|------|
| Webhooks (recommended) | Real-time processing needed | Immediate notification; no polling overhead; scales well | Requires HTTPS endpoint; signature verification needed |
| Polling | Webhooks unavailable; batch processing OK | Simpler setup; no external endpoint required | Higher latency; more API calls; harder to scale |

### When to sync engagement vs opportunity

| Entity | Sync When | Includes |
|--------|-----------|----------|
| Engagement | Appointment scheduled; rep will record | Time, location, rep, linked account/opportunity |
| Opportunity | Deal created or updated; need deal context | Amount, disposition, customer, rep |
| Both | Typical flow | Engagement links to Opportunity; recording context is rich |

## Workflow

### 1. Build a CRM integration (bidirectional sync)

1. **Get org API token:** Log into Siro dashboard → Person Icon → API Tokens → Generate New API Token. Store securely.

2. **Create OAuth app:** POST `/v1/core/oauth/apps` with `appName` and `owner` (Siro User UUID). Store `clientId` and `clientSecret`.

3. **Generate OAuth access token:** POST `/v1/core/oauth/apps/{clientId}/access-token` with `clientSecret`, `userId`, and `scope: "read"`. Token expires in 16 hours; regenerate as needed.

4. **Sync engagements:** PUT `/v1/integrations/sync/engagements` with appointment data (externalId, startTime, endTime, subject, engagementUsers, account, opportunity). Repeat for each appointment.

5. **Sync opportunities:** PUT `/v1/integrations/sync/opportunities` with deal data (externalId, name, amount, disposition, account, opportunityUsers).

6. **Subscribe to webhooks:** Siro dashboard → Organization Admin → Webhooks → Add Endpoint. Provide HTTPS URL, select `integrations.recordingProcessed` and `integrations.recordingLinked`. Copy signing secret.

7. **Verify webhook signatures:** Use Svix library (Node.js, Python, Go) to verify `svix-id`, `svix-timestamp`, `svix-signature` headers. Act only when both `recordingProcessed` and `recordingLinked` are `true`.

8. **Fetch recording details:** GET `/v1/core/recordings/{recordingId}?showSummary=true&showEntityExtractions=true` (base URL: `https://api.siro.ai`) with OAuth token. Extract summary and entity extractions.

9. **Write back to CRM:** Use webhook payload CRM IDs (`crm.engagement.externalId`, `crm.opportunity.externalId`) to identify the CRM record. Attach summary, extractions, and scorecard metrics.

### 2. Upload phone recordings

1. **Get OAuth access token:** Follow steps 2–3 above with `scope: "write"`.

2. **Get signed upload URL:** POST `/v1/core/recordings/signed-urls` (base URL: `https://api.siro.ai`) with `fileSizeInBytes`, `fileType` (mp3, aac, wav). Receive `uploadUrl` and `downloadUrl`.

3. **Upload audio:** PUT to `uploadUrl` with raw audio file. Wait for 2xx response.

4. **Create recording:** POST `/v1/core/recordings/upload` with `fileUrl` (use `downloadUrl` from step 2), `fileType`, `userId` or `userEmail` or `userPhone`, `dateCreated` (actual call time), and optional `integrationConnectionId` + `crmObjects` for CRM linking.

5. **Verify in Siro:** Recording appears in Siro within 30 minutes; linked to rep and CRM record if provided.

### 3. Trigger recordings from your app

1. **Get conversation type UUIDs:** GET `/v1/core/conversation-configurations?organizationId={orgId}` (base URL: `https://app.siro.ai/api-internalApi`) with Siro auth token. Extract `conversationType` UUIDs.

2. **Build deep link:** Construct `https://app.siro.ai/record?action=start&title={title}&conversationType={uuid}&redirectUrl={encoded-url}`.

3. **Embed in your app:** Use as href or deep link handler. Siro app opens, starts recording, then redirects back to your app.

4. **Optional CRM linking:** Add `integrationConnectionId` and URL-encoded `crmObjects` array to link recording to CRM entities at start time.

## Common gotchas

- **Webhook signature verification is mandatory.** Use the Svix library; don't parse JSON before verifying. Raw body only.

- **Both webhook events must be `true` before acting.** Events fire independently and may arrive in either order. Dedupe by `recordingId` if downstream side-effects aren't idempotent.

- **OAuth tokens expire after 16 hours.** Regenerate proactively; don't wait for 401 errors.

- **`externalId` vs `id` confusion.** `externalId` is your CRM's ID; `id` is Siro's internal UUID. When targeting entities by `externalId`, also send `integrationConnectionId` so Siro knows which CRM connection.

- **User mapping is email-based by default.** When syncing engagements, include `email` on `engagementUsers`. Siro matches External Users to Siro Users by email. Manual override in Settings → Integrations → User Mapping if needed.

- **Engagement syncing is the core integration.** Opportunities add context, but engagements are what trigger recording linking. Always sync engagements for appointments you want to record.

- **Phone recording upload requires `userId`, `userEmail`, or `userPhone`.** At least one must be present; otherwise the recording won't ingest. Use `userId` (Siro User UUID) when possible for most reliable linking.

- **CRM object IDs must already exist in Siro.** When linking phone recordings via `crmObjects`, the object IDs must have been synced (via native CRM integration or API sync) before the recording is uploaded. Siro won't create new CRM records.

- **Deep link `conversationType` UUID is required for pre-selection.** If the UUID is missing or malformed, the recording falls back to the rep's default conversation type. No error is shown.

- **Rate limit is 1000 requests per token per minute.** Batch sync operations; don't make one request per record.

## Verification checklist

Before submitting work:

- [ ] Organization API token is stored securely and not logged or exposed
- [ ] OAuth app `clientSecret` is stored securely; never committed to version control
- [ ] OAuth access tokens are regenerated before 16-hour expiry (or handled gracefully on 401)
- [ ] Webhook endpoint is HTTPS and publicly accessible
- [ ] Webhook signature verification uses Svix library and raw request body
- [ ] Webhook handler checks both `recordingProcessed` and `recordingLinked` before processing
- [ ] Webhook handler dedupes by `recordingId` if side-effects aren't idempotent
- [ ] Engagements are synced with `externalId`, `startTime`, `endTime`, and at least one `engagementUser`
- [ ] Opportunities are synced with `externalId`, `amount`, and `disposition`
- [ ] User emails in sync payloads match Siro User emails for automatic mapping
- [ ] Phone recordings include `userId` or `userEmail` or `userPhone` for rep attribution
- [ ] Phone recordings include `dateCreated` (actual call time, not upload time)
- [ ] CRM object IDs in phone recording uploads already exist in Siro
- [ ] Deep links use URL-encoded `crmObjects` array and include `integrationConnectionId`
- [ ] Recording details are fetched with both `showSummary=true` and `showEntityExtractions=true` query params
- [ ] Entity extractions are written back to CRM using `crmFieldName` mappings from the response
- [ ] Test with a single engagement, record a conversation, verify CRM context appears in Siro

## Resources

**Comprehensive page listing:** https://docs.siro.ai/llms.txt

**Critical documentation:**
- [Getting Started: Building a Custom API CRM Integration](https://docs.siro.ai/getting-started) — end-to-end bidirectional sync workflow
- [Phone Integration Getting Started](https://docs.siro.ai/phones-integration-getting-started) — uploading phone recordings from VoIP systems
- [Available Integrations](https://docs.siro.ai/Integrations/Available_Integrations) — pre-built integrations (Salesforce, HubSpot, Pipedrive, etc.)

---

> For additional documentation and navigation, see: https://docs.siro.ai/llms.txt