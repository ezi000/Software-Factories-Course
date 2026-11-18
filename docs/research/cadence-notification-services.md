# Managed notification services for Cadence

Research date: **2026-11-18**. Resolution asset for [Research managed notification services for Cadence](https://github.com/ezi000/Software-Factories-Course/issues/3), under [Map Cadence notifications](https://github.com/ezi000/Software-Factories-Course/issues/1). This is evidence and an advisory recommendation; **the human has not selected a provider**. Throwaway branch: `research/cadence-notification-services`; do not merge.

## Recommendation to take into selection

Evaluate **Knock first for the Notification Feed**, with **Courier as the strongest commercial alternative**. Knock offers a direct fit for feed-first React UI, flexible headless hooks, documented browser FCM integration, and delivery retries. Courier deserves equal consideration if the paid-entry cost or multi-channel billing dominates. **Novu Cloud** has an attractive entry price and an open-source migration option, but its documented lack of automatic downstream provider retries and organization-gated idempotency increase integration responsibility. These are assessments based on the evidence below, not a purchase decision.

Do not select on a generic “push supported” checkbox. All three can participate in browser push through another delivery provider; the web client still needs permission, registration, a service worker, and secure per-user registration ownership. No accounts, subscriptions, delivery tests, or production changes were made for this research.

## Fit for the feed

| Concern | Knock | Novu Cloud | Courier |
| --- | --- | --- | --- |
| Persistent Notification Feed | Hosted feed, unread/read/seen/archive state; React feed and inbox components | Hosted Inbox; read/unread/archive actions, counts and real-time updates | Hosted Inbox with persisted messages, read/archive sync across devices, unread counts and pagination |
| React / custom UI | `@knocklabs/react`, popover and inbox; React headless hooks and JS API | `@novu/react`; current `@novu/js` for custom headless clients; avoid legacy `@novu/notification-center` | React SDK and web components; custom interface can reuse SDK messages, counts and actions |
| Cadence layout | Suitable for sidebar launcher, panel or full page | Inbox content can be placed in custom drawer or page | Prebuilt or custom inbox can fit either layout |

Sources: [Knock React feed](https://docs.knock.app/in-app-ui/react/feed), [Knock headless feed](https://docs.knock.app/in-app-ui/react/headless/feed), [Knock message states](https://docs.knock.app/send-notifications/message-statuses), [Novu migration/current SDKs](https://docs.novu.co/platform/inbox/migration-guide), [Novu custom popover](https://docs.novu.co/platform/inbox/advanced-customization/customize-popover), [Courier Inbox](https://www.courier.com/docs/in-app/inbox.md). Layout suitability is an inference; visual fit and keyboard behavior still need the human-reviewed prototype.

Persistence does not mean indefinite contractual retention. Knock currently lists unlimited configurable feed retention. Novu separates operational Activity Feed retention from end-user Inbox retention; its pricing matrix lists 30 days for Free Inbox and 90 days for Pro/Team, versus 24 hours/7 days/90 days for activity logs. Courier's pricing lists log retention, which must not be mistaken for an Inbox retention promise; obtain its exact Inbox expiration/deletion policy before selecting. The extracted Novu table loses some column alignment, so reconfirm the Inbox limits in the account/quote. Sources: [Knock pricing](https://knock.app/pricing), [Novu pricing](https://novu.co/pricing/), [Courier pricing](https://www.courier.com/pricing).

## Browser push and auxiliary email

- **Knock:** official FCM integration explicitly covers mobile **and web**. Register the browser with Firebase, send its token to the app server, and store it in recipient channel data. Knock accepts `webpush` overrides. Token deregistration after invalid-token bounces is opt-in; application logout/opt-out cleanup remains necessary. [Knock Firebase integration](https://docs.knock.app/integrations/push/firebase)
- **Novu:** FCM explicitly supports web; its push integrations also include OneSignal and Pushpad. A managed Novu account is an orchestration layer over separately configured push credentials and subscriber registrations. [Novu FCM](https://www.novu.co/integrations/fcm/), [Novu push integrations](https://docs.novu.co/platform/integrations/push)
- **Courier:** first-party integration documentation advertises mobile and web push with Firebase/OneSignal. Its FCM integration accepts stored recipient tokens and provider overrides. The detailed SDK setup is heavily mobile-oriented; browser registration, foreground/background handling, and logout behavior need a browser-specific proof before commitment. Do not infer those from the mobile SDK. [Courier integrations](https://www.courier.com/integrations), [Courier FCM setup](https://www.courier.com/docs/integrations/push/firebase-fcm.md)

FCM web requires HTTPS, a service worker, Web Push credentials and user permission. Current Firebase guidance now recommends installation IDs while these orchestration docs describe FCM registration tokens. That is a real version-compatibility question: pin and verify the supported Firebase registration path with the chosen provider rather than assuming an installation ID can replace a token. [Firebase web setup](https://firebase.google.com/docs/cloud-messaging/web/get-started)

For all three, email is auxiliary delivery with a separately configured production sender such as SES, Postmark, Resend or SendGrid. Budget sender fees and domain/deliverability setup separately. Knock's own email integration is testing-only; Novu connects third-party channel credentials; Courier explicitly uses customer provider accounts. [Knock email](https://docs.knock.app/integrations/email/overview), [Novu integrations](https://docs.novu.co/platform/concepts/integrations), [Courier first email](https://www.courier.com/docs/guides/send-your-first-email.md)

**Proposed product boundary:** keep in-app as the durable record; treat browser push as an optional alert and email as a selected delayed reminder. Browser permission denial must leave the feed useful. A provider accepting a push does not establish that the person saw it. The delay, eligibility check and cancellation when read must be designed explicitly, not assumed from a multi-channel template.

## Preferences and identity isolation

| Provider | Preferences | Browser authentication requirement |
| --- | --- | --- |
| Knock | User PreferenceSet supports channels, categories and workflows; evaluated during workflow execution | Enable enhanced security; require server-signed user JWT. Public key alone can otherwise impersonate any user |
| Novu | Global and per-workflow channel preferences; more specific settings win; critical workflows bypass subscriber preferences | Enable HMAC security in the integration; compute subscriber hash on server. Subscriber ID plus public application ID alone is unsafe |
| Courier | Topics/sections, channel choices, tenant defaults and user choices; hosted or embedded preferences | Backend calls token-issue API for scoped expiring JWT; scope to `user_id` and inbox permissions; add preferences scopes only when needed |

Sources: [Knock preferences](https://docs.knock.app/concepts/preferences), [Knock security](https://docs.knock.app/in-app-ui/security-and-authentication), [Novu preferences](https://docs.novu.co/platform/concepts/preferences.md), [Novu production security](https://docs.novu.co/platform/inbox/prepare-for-production), [Courier preferences](https://www.courier.com/docs/recipients/preferences/overview.md), [Courier authentication](https://www.courier.com/docs/in-app/authenticate-users.md).

Cadence-specific design implications (recommendations, not implemented):

1. Derive identity from the server session, then consistently convert numeric IDs to strings, e.g. `String(userId)`. Do not mint credentials for a browser-submitted arbitrary user ID. Use separate provider environments for development and production; resettable seed IDs must never share production identities.
2. On DevUI switching/logout, immediately hide the old feed/count, tear down sockets, clear identity-specific caches, discard in-flight results and mount/authenticate the new identity. Courier explicitly supplies sign-out; verify equivalent lifecycle behavior for the selected SDK. Country switching must not change identity.
3. Browser push registration belongs to a browser installation, not inherently a person. Remove the old user's registration association before attaching the same browser to the new user. Unmounting a feed does not revoke an old push registration. Test multiple tabs and delayed responses.
4. Authenticate every destination route again. A refund decision may revoke learning access; a historical notification must not grant access. Avoid putting private question or refund details into OS-visible push previews.
5. Current DevUI is development impersonation, not production authentication. Provider JWT/HMAC protects its API only as well as the session that authorizes issuance.

Repository evidence at base `994a21e054f0ae8e6057179241441ba336f33394`: [session implementation](https://github.com/ezi000/Software-Factories-Course/blob/994a21e054f0ae8e6057179241441ba336f33394/app/lib/session.ts) returns numeric user IDs and uses a development cookie secret; [DevUI](https://github.com/ezi000/Software-Factories-Course/blob/994a21e054f0ae8e6057179241441ba336f33394/app/components/dev-ui.tsx) switches users by server form action; [package manifest](https://github.com/ezi000/Software-Factories-Course/blob/994a21e054f0ae8e6057179241441ba336f33394/package.json) uses React Router and React. SDK peer compatibility and SSR/hydration have not been exercised.

## Event delivery and recovery

| Layer | Knock | Novu Cloud | Courier |
| --- | --- | --- | --- |
| Server event API | Workflow trigger with recipient IDs, actor and data | Trigger workflow for subscribers with payload | Send template/data to user ID; journeys for multi-step delivery |
| Trigger deduplication | `Idempotency-Key` on workflow trigger; successful responses cached 24h; mismatched parameters rejected | API idempotency requires organization enablement; POST/PATCH with API key; success/error cache 24h; in-progress 409. `transactionId` is not equivalent | Supported POST endpoints, including send; endpoint-scoped key; default replay 25h, optionally longer |
| Downstream retries | Retryable failures are re-enqueued; logs and manual retry for undelivered messages | **Provider send attempted once**, including 429/503 failures; no automatic provider failover. Subscribe to failure events and design recovery | Transient provider failures retried; provider/channel failover. Docs describe about 24h retry window and configurable overall message timeout |

Sources: [Knock trigger](https://docs.knock.app/send-notifications/triggering-workflows/api), [Knock API idempotency](https://docs.knock.app/api-reference/overview), [Knock delivery](https://docs.knock.app/send-notifications/delivering-notifications), [Novu idempotency](https://docs.novu.co/api-reference/idempotency.md), [Novu retry policy](https://docs.novu.co/platform/developer/delivery-retries.md), [Courier API conventions](https://www.courier.com/docs/reference/api-overview.md), [Courier routing and failover](https://www.courier.com/docs/send/routing.md).

Courier caveat: generic `push` routing can also select Inbox. Name the external push provider explicitly when implementing fallback, or a successful Inbox write may prevent the intended email fallback. Provider error failover also differs from responding to a later email bounce. [Courier routing](https://www.courier.com/docs/send/routing.md)

Recommended architecture for every candidate: commit a durable notification intent with the business transaction, dispatch from a retryable outbox, and persist its stable event/recipient key and provider request ID. Retry submission with the same idempotency key. Do not send from a React effect or an uncommitted action. Provider windows are finite; maintain an application deduplication/reconciliation policy beyond them. This is an architectural inference from the documented API boundaries, not an existing Cadence capability or an exactly-once guarantee.

Model candidate event families separately: question created, reply created, Refund Request submitted, Refund Request decided, Team Seat claimed. Have the catalog decision determine recipients and self-notification suppression. A notification read/archive action must not itself approve a Refund Request or claim a seat. Do not introduce “refund paid” without a verified payout event. Store stable entity identifiers and links; leave business authorization and outcomes in Cadence.

## Cost snapshot and assumptions

USD monthly public rates observed **2026-11-18**, excluding tax, delivery-provider charges, negotiated discounts and optional AI/add-ons. No actual Cadence usage forecast exists.

| Service | Free tier | Entry paid tier / meter |
| --- | --- | --- |
| Knock | 10,000 messages | Starter $250 includes 50,000; then $0.005/message. A message is one successful recipient-channel delivery |
| Novu Cloud | 10,000 workflow runs; current directly opened page says 5 workflows | Pro from $30 includes 30,000 runs; $1.20/1,000 additional. One run per subscriber regardless of channels |
| Courier | Developer 10,000 sends | Business $0.005 for **every** send, including first 10,000. One template/recipient send may cover multiple channels |

Sources: [Knock prices](https://knock.app/pricing), [Novu prices](https://novu.co/pricing/), [Novu billing](https://docs.novu.co/platform/account/billing.md), [Courier billing](https://www.courier.com/docs/workspaces/billing.md). Search snippets showed an older Novu Free workflow count of 20; use the directly opened current page's 5 and reconfirm at signup.

Illustrative arithmetic, **not a quote**: 20,000 recipient-events/month, one workflow/template per event, all receive in-app plus email, no additional reminders, retries or batching. Knock counts 40,000 messages: **$250**. Novu counts 20,000 runs: **$30**. Courier counts 20,000 sends: **$100**. At 100,000 such events: Knock 200,000 messages = **$1,000**; Novu Pro 100,000 runs = **$114** at published overage; Courier 100,000 sends = **$500**. Feature requirements could force higher plans.

Fan-out multiplies recipient-events. Multiple independently triggered reminders/templates can increase bills even for one business event. Novu and Courier count test-environment usage; Courier also counts preference-blocked and undelivered sends. Novu Free stops new runs at its cap (402). These materially affect budget and outage handling. [Novu billing](https://docs.novu.co/platform/account/billing.md), [Courier billing](https://www.courier.com/docs/workspaces/billing.md)

## Lock-in, open questions and acceptance gates

All three own provider-specific templates, workflow configuration, hosted read state and preference models. Changing email transport is easier than migrating the orchestration/feed service. Novu's open-source option is useful, but does not prove that Cloud features, data exports or migrations are automatic. [Novu hosting/billing distinctions](https://docs.novu.co/platform/account/billing.md)

Recommended portability boundary: Cadence owns event names, recipient calculation, stable IDs, destination rules and the outbox; keep provider SDK calls behind one server adapter. Export/version templates and preference mappings where possible. Establish history/read-state export, deletion and migration procedures before relying on a provider as the sole long-term history store.

Questions for the human selection and later spec:

- What monthly volume, acceptable paid floor and feed-history duration are required? Which team member owns accounts and billing?
- Is browser push in the first release, and which browsers/devices must pass? Does the Firebase registration API version fit the chosen orchestrator today?
- Which topics/channels can be disabled, which are mandatory, and what delay/read check suppresses email? Do not silently label refund decisions “critical.”
- What reliability target justifies an outbox, reconciliation and manual replay? For Novu, is custom downstream recovery acceptable, and is idempotency enabled on the account?
- Which data region, log/history retention, deletion/export behavior and support/SLA tier meet the project's actual needs?

Before implementation acceptance, exercise two-user isolation and DevUI switching, expired credentials, cross-tab counts, offline reconnect, read/unread failures, duplicate trigger replay, provider failure and rate limiting, permission denied/revoked, service-worker delivery with the tab closed, shared-browser sign-out, and links after access removal. These are proposed validation gates; no live provider verification was performed.

## Research verification

Read official vendor docs/pricing and first-party Firebase documentation; used direct Markdown HTTP reads where the browser fetch failed. Avoided competitor claims as evidence about competitors. Reviewed the repository identity/session files. No production app edits or dependency installation; therefore the app typecheck/test requirement does not apply. Artifact scope and whitespace were checked with Git before commit. Provider plans and documentation are mutable and should be rechecked at selection.
