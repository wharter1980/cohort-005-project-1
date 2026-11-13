# Hosted notification vendors for Cadence

Research for [#2](https://github.com/wharter1980/cohort-005-project-1/issues/2), part of map [#1](https://github.com/wharter1980/cohort-005-project-1/issues/1). Researched 2026-11-13 from vendor docs, pricing pages and the npm registry. Vendors change their pricing pages often, so check the numbers again before signing anything.

**Question.** Which hosted vendors (Knock, Novu Cloud, Courier, SuprSend, MagicBell) fit Cadence? They are compared on: a React in-app feed (prebuilt and headless), email and browser web push from the same workflow, per-type and per-Channel preferences with email unsubscribe, a Node server SDK, secure feed auth, email providers, identity sync, and free vs production pricing. Self-hosting is out.

**Cadence context.** React Router (framework mode; `package.json` pins `react-router` 8.3.0), React 19.2, a Node server, SQLite/Drizzle, Tailwind/shadcn. Every vendor's Node SDK is a plain server-side HTTP client, so all of them can be called from loaders and actions. That makes the Node SDK a tie, not something that separates the vendors.

## Summary table

| | **Knock** | **Novu Cloud** | **Courier** | **SuprSend** | **MagicBell** |
|---|---|---|---|---|---|
| React feed, prebuilt | `@knocklabs/react`: `NotificationFeedPopover`, `NotificationFeed`, `NotificationIconButton` [1] | `@novu/react` `<Inbox />` (Bell, Notifications, Preferences) [8] | `@trycourier/courier-react` v9 `<CourierInbox />` [14] | `@suprsend/react` drop-in components [17] | `@magicbell/magicbell-react` [22] |
| React feed, headless | Hooks in `@knocklabs/react` and `@knocklabs/client` [1] | Headless hooks in `@novu/react`, plus `@novu/js` [8] | `useCourier` hook and the client [14] | Headless implementation documented [17] | `@magicbell/react-headless` [npm] |
| React 19 peer dep (npm) | yes (`^17 \|\| ^18 \|\| ^19`) | yes (`^18 \|\| ^19`) | yes (`>=18`) | yes (`>=16.8`) | not checked |
| Browser web push | **Through FCM only** (Firebase SDK and service worker in the app) [3][4] | Through FCM, OneSignal, Pushpad, Pusher Beams or Push Webhook; no native VAPID [9] | Through FCM or OneSignal; "Courier Push" is in-app WebSocket only, not OS push [15] | **Native VAPID**; SuprSend holds the keys and the SDK provides the service worker [19] | **Native VAPID** (you supply the key pair) [23] |
| Preference model | Workflow, category, channel-type and channel; all must be true; defaults true; conditions [5] | Workflow defaults, then global, then per-workflow channel; "critical" workflows can't be changed by users [10] | Subscription topics (`OPTED_IN`, `OPTED_OUT`, `REQUIRED`) plus per-topic channel routing [16] | Category or sub-category, global channel and per-category channel opt-out; "Can't Unsubscribe" with mandatory channels [18] | Category times channel `enabled` flags [24] |
| Mandatory Notifications | Workflow setting to override preferences (not verified in this pass) | `critical` workflow [10] | `REQUIRED` topic [16] | "Can't Unsubscribe" category [18] | not documented |
| Email unsubscribe | Hosted preference page (signed URL); one-click unsubscribe headers and footer link for *commercial* workflows [5][6] | **None by default.** You build a tokenized link and call the preference API [11] | Hosted page plus `unsubscribe` variable; List-Unsubscribe headers sent **only through SendGrid** [16][16b] | Hosted preference page; `List-Unsubscribe` and `List-Unsubscribe-Post` headers on "commercial" sub-categories [18] | No email unsubscribe docs found [24] |
| Feed auth | RS256 JWT signed with your private signing key; Node `signUserToken`; "enhanced security mode" [2] | HMAC-SHA256(secretKey, subscriberId) `subscriberHash`; **never expires** [12] | Short-lived scoped JWT minted by Courier `POST /auth/issue-token` [13] | ES256 JWT signed with your signing key; `refreshUserToken` callback [20] | HS256 JWT signed with the project secret [25] |
| Email providers | SES, Cloudflare, MailerSend, Mailgun, Mailjet, Mailtrap, Mandrill, Postmark, Resend, SendGrid, SMTP, SparkPost, plus a Knock test sender [7] | 19 providers including SES, Resend, Postmark, SendGrid, Mailgun, SMTP; demo provider for tests [9b] | Broad (SendGrid, SES, Postmark, Mailgun and others; list not re-fetched) | Not re-fetched in this pass | SES, Mailgun, SendGrid, SMTP, Ping Email (5) [22] |
| Identity sync | Recipients identified inline on trigger, or upserted with `users.identify` | Subscribers created inline on trigger, or upserted with `@novu/api` | Profiles inline on send, or through the profiles API | `user` upsert in Node SDK; entity `distinct_id` | Users upserted through the API |
| Node SDK (npm, Nov 2026) | `@knocklabs/node` 1.41.1 (updated 2026-11-10) | `@novu/api` 3.19.1 (`@novu/node` is **deprecated**) | `@trycourier/courier` 10.0.0 | `@suprsend/node-sdk` 1.15.1 | `magicbell` 4.5.1 |
| **Free tier** | 10,000 messages a month; all features including feeds, preferences and unlimited channels; 99.9% SLA [P1] | 10K workflow runs a month, **5 workflows**, 3 team members, 24h activity retention [P2] | 10,000 sends a month (one send to several channels counts once) [P3] | 10,000 notifications a month, **but no preferences** (preference rows are "—" on Free and Essentials) [P4] | **1,000 deliveries** a month, counted per channel [P5] |
| **Production price** | Starter **$250/mo** for 50K messages, then $0.005 each [P1] | Pro **from $30/mo** for 30K runs, then $1.20 per 1K; Team from $250/mo for 250K [P2] | Business pay-as-you-go **$0.005/send**, including the first 10K; digest and batching are **Enterprise only** [P3] | Business **$250/mo** (billed annually) for 50K is the cheapest plan with preferences; Essentials at $100/mo has none [P4] | Startup **$249/mo** for 50K deliveries, then $0.0025 each [P5] |

## How each vendor fits Cadence

### Knock: best fit overall
- Strengths: the free tier has every feature. Feeds, preferences and channels are not gated, so a production-shaped prototype costs nothing [P1]. The preference model maps directly to "per-type, per-Channel": workflow or category crossed with channel type [5]. Feed auth is a JWT with an expiry, signed by a dedicated Node helper, which fits loaders well: sign in the root loader and pass it to the client [2]. It has the widest list of email providers, including Resend and Postmark [7], and a React package with both a prebuilt popover and hooks that declares React 19 [1].
- Weaknesses:
  - Web push isn't first-class. It goes through FCM, so Cadence would need a Firebase project, the Firebase JS SDK and a service worker [3][4].
  - The one-click unsubscribe header is built for *commercial* messages [6]. Per-type opt-out for transactional email goes through the hosted preference page or Cadence's own settings UI [5].
  - Pricing jumps from $0 to $250/mo once volume passes 10K messages a month [P1].

### Novu Cloud: cheapest paid path
- Strengths:
  - Pro is $30/mo [P2].
  - `<Inbox />` includes a Preferences view, and there are headless hooks [8].
  - `critical` workflows cleanly model mandatory Notifications [10].
  - It has many email providers [9b].
- Weaknesses:
  - There is no email unsubscribe by default, so Cadence would build tokenized links itself [11].
  - Feed auth is an HMAC with no expiry [12].
  - Web push needs a third-party provider [9].
  - The free tier caps you at 5 workflows and keeps activity for only 24h, which is tight for a catalogue of Notifications [P2].
  - The old Node SDK is deprecated, so use `@novu/api`.

### Courier: capable but awkward for this shape
- Strengths:
  - The free tier is generous, and pay-as-you-go pricing is cheap [P3].
  - It has a `REQUIRED` topic type, a hosted preference page and short-lived scoped JWTs [13][16].
- Weaknesses:
  - Digest and batching are Enterprise only [P3].
  - List-Unsubscribe headers are sent only through SendGrid [16b].
  - Some preference APIs return `402` on lower plans [16].
  - Web push goes through OneSignal or FCM [15].

### SuprSend: best channel and unsubscribe features, but preferences cost money
- Strengths:
  - It is the only candidate with native VAPID web push where the vendor manages the keys and the SDK provides the service worker [19].
  - It has the strongest built-in email unsubscribe: RFC 8058 headers and a hosted preference page [18].
  - It has ES256 JWTs with a refresh callback [20].
- Weakness: **preferences aren't included on Free or Essentials** [P4]. The free tier can't exercise the preference requirement, and production starts at $250/mo (billed annually).

### MagicBell: weakest fit
- Strength: native web push [23].
- Weaknesses:
  - The free tier has only 1,000 deliveries, counted per channel [P5].
  - It supports only 5 email providers [22].
  - The model is broadcast-centric, with thin documentation on workflows, mandatory types and unsubscribe [24].
  - The JWT examples use one-year expiries [25].

## Recommendation (for the human decision in #5)

**This research recommends Knock**, with **Novu Cloud** as the runner-up if cost is the main concern.

Reasons:
1. It meets every hard requirement on the free tier, preferences included. SuprSend doesn't (preferences are paid) and MagicBell's free tier is too small.
2. Its preference model fits per-type and per-Channel directly, and it has a hosted preference page for email links.
3. It has the most production-ready feed auth story for a React Router server: a short-lived RS256 JWT signed in a loader.
4. Its React SDK has both prebuilt and headless options and declares React 19, which keeps the open question of "prebuilt feed vs our own shadcn UI" open.
5. It has the broadest list of email providers, so the email provider can be chosen separately later.

**Accepted costs:**
- Web push needs Firebase (FCM) set up in the browser.
- Production pricing is $250/mo above 10K messages a month.
- Transactional per-type email opt-out goes through the preference page or our own settings UI, not RFC 8058 headers.

**What would change the pick:**
- If browser web push becomes a priority and adding Firebase isn't acceptable, SuprSend is the strongest on that axis, if the $250/mo Business plan is acceptable.
- If production cost dominates, Novu Pro at $30/mo, accepting a home-built unsubscribe link and HMAC feed auth.

## Gaps (not verified in this pass)
- Knock's exact header output (does it send `List-Unsubscribe-Post`?) and its per-workflow "override preferences" setting for mandatory types.
- Whether Novu's `<Inbox />` and `<Preferences />` components are gated by plan (the pricing table was ambiguous) [P2].
- Full email provider lists for Courier and SuprSend.
- MagicBell's React 19 support and its service worker and Safari details.
- The identity sync rows come from each vendor's SDK and API model and were not re-fetched one by one.

## Sources
- [1] Knock React SDK overview: https://docs.knock.app/in-app-ui/react/overview
- [2] Knock in-app security and authentication: https://docs.knock.app/in-app-ui/security-and-authentication
- [3] Knock push overview: https://docs.knock.app/integrations/push/overview
- [4] Knock FCM (web push): https://docs.knock.app/integrations/push/firebase, https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
- [5] Knock preferences overview: https://docs.knock.app/preferences/overview
- [6] Knock commercial unsubscribe: https://docs.knock.app/preferences/commercial-unsubscribe
- [7] Knock email integrations: https://docs.knock.app/integrations/email/overview
- [8] Novu Inbox overview: https://docs.novu.co/platform/inbox/overview
- [9] Novu push integrations: https://docs.novu.co/platform/integrations/push
- [9b] Novu email integrations: https://docs.novu.co/platform/integrations/email
- [10] Novu preferences: https://docs.novu.co/platform/concepts/preferences
- [11] Novu unsubscribe guidance (email integration docs): https://docs.novu.co/platform/integrations/email
- [12] Novu Inbox production (HMAC): https://docs.novu.co/platform/inbox/prepare-for-production
- [13] Courier Inbox authentication: https://www.courier.com/docs/platform/inbox/authentication
- [14] Courier Inbox overview: https://www.courier.com/docs/platform/inbox/inbox-overview
- [15] Courier push overview: https://www.courier.com/docs/external-integrations/push/intro-to-push
- [16] Courier preferences overview: https://www.courier.com/docs/platform/preferences/preferences-overview
- [16b] Courier hosted preference page and headers: https://courier.com/docs/platform/preferences/hosted-page
- [17] SuprSend React inbox: https://docs.suprsend.com/docs/inbox-react
- [18] SuprSend user preferences: https://docs.suprsend.com/docs/user-preferences
- [19] SuprSend web push: https://docs.suprsend.com/docs/web-push-quick-start, https://docs.suprsend.com/docs/js-webpush
- [20] SuprSend client authentication: https://docs.suprsend.com/docs/client-authentication
- [22] MagicBell docs home: https://www.magicbell.com/docs
- [23] MagicBell web push channel: https://www.magicbell.com/docs/channels/web_push
- [24] MagicBell in-app channel and primitives: https://www.magicbell.com/docs/channels/in_app, https://www.magicbell.com/docs/primitive
- [25] MagicBell user JWT: https://www.magicbell.com/docs/api/authentication/user
- [P1] https://knock.app/pricing · [P2] https://novu.co/pricing · [P3] https://www.courier.com/pricing · [P4] https://www.suprsend.com/pricing · [P5] https://www.magicbell.com/pricing
- [npm] `npm view <pkg> version peerDependencies` run 2026-11-13 for each package named above.
