# Which notification services fit Cadence?

Research for [#2](https://github.com/clearwater1968/cohort-005-project-fork/issues/2), part of the map [#1 Notifications in Cadence](https://github.com/clearwater1968/cohort-005-project-fork/issues/1). Researched 2026-11-18 against primary sources: provider docs, pricing pages, official SDK repos and npm registry metadata. Every claim links to its source. Items marked **UNVERIFIED** could not be confirmed from a primary source.

This document ranks the options. It **does not pick one**. The pick belongs to the provider grilling ticket.

## Cadence's assumptions

- React Router 8.3, React 19, SSR, Tailwind v4, `radix-ui`, a Node server, and a cookie session (`app/lib/session.ts`) whose user the DevUI switches.
- Scale used for pricing: about **2,000 monthly active recipients** and **20,000–50,000 Notifications a month**. Each Notification goes to the Notification Feed, and a few also go to web push.
- Repo note: the map says email goes through Postmark in `app/lib/mailer.server.ts`, but on `main` today that file and any Postmark dependency are missing. The "take over from Postmark" requirement is about the future, not the current code.

## Comparison table

| | Knock | Novu Cloud | Courier | MagicBell | SuprSend | OneSignal |
|---|---|---|---|---|---|---|
| **Headless feed API + React** | Yes: `@knocklabs/react` components and hooks | Yes: `@novu/react` `<Inbox/>` and hooks | Yes, but components are shadow-DOM web components; headless via `useCourier()` | Components in `@magicbell/react`; headless only via `magicbell-js` (hooks package looks legacy) | Yes: `@suprsend/react` drop-in, `@suprsend/react-core` hooks | **No web feed** |
| **Tailwind/Radix fit** | Good (hooks, or skip their CSS) | Best (`appearance.elements` takes Tailwind classes; hooks) | Weak for drop-in (shadow DOM); fine headless | Medium (prebuilt CSS and overrides) | Good headless; drop-in uses Emotion and theme objects | n/a |
| **Browser web push** | Via FCM only | Via FCM, OneSignal, Pushpad or Pusher Beams | Via Pusher Beams only (documented) | **Native** (VAPID, own SW, `WebPushButton`) | **Native** (VAPID, own SW, `registerPush()`) | **Native**, best prompts |
| **Email, Postmark supported** | Yes, Postmark | Yes, Postmark | Yes, Postmark | No Postmark (SES, Mailgun, SendGrid, SMTP) | Yes, Postmark | Own email, SendGrid or Mailgun; no Postmark |
| **Node SDK** | `@knocklabs/node` (TS, ESM) | `@novu/api` (TS, ESM) | `@trycourier/courier` (TS, CJS and ESM) | `magicbell-js` (TS, CJS and ESM) | `@suprsend/node-sdk` (types, **CJS only**) | `@onesignal/node-onesignal` (CJS and types) |
| **Server-signed feed auth** | RS256 JWT (`signUserToken`), optional per env | HMAC `subscriberHash`, optional per env, never expires | JWT from `/auth/issue-token` (scoped, expiring) | HS256 user JWT | ES256 JWT, optional per workspace | **Not supported on Web SDK** |
| **Dev/prod separation** | Dev and Prod environments with promotion | Dev and Prod environments with publish | Test and Production environments | Separate projects (1 on free) | Sandbox, Staging and Prod workspaces | Separate apps |
| **Free tier** | 10k messages/mo (counted per channel) | 10k runs/mo, **24 h feed retention** | 10k sends/mo | 1k deliveries/mo | 10k notifications/mo, no prefs | Web push free; 10k emails/mo |
| **Cheapest fit at our scale** | Starter **$250–500/mo** | Pro **$30–54/mo** (7-day retention); Team $250 (90-day) | Business **≈$100–250/mo** | Startup **$249–374/mo** | Essentials **$100/mo** (no prefs); Business $250 (prefs) | ≈$0–27/mo |
| **Pricing unit** | Message = 1 user × 1 channel | Workflow run (channels not multiplied) | Send (channels not multiplied) | Delivery = per channel | Notification | Push per subscriber, email per send |
| **Preferences** | Rich model; hosted page; no React component | Per channel/workflow; built into Inbox; `usePreferences` | Per topic × channel; hosted page and component | Per channel/category; `UserPreferences` component | Category × channel; ready UI on Business+ | Push tags only |
| **Real-time feed** | WebSocket | WebSocket | WebSocket | WebSocket | Socket | n/a |
| **Data region** | US only (EU "expected by end of 2026") | US or EU (Frankfurt), all plans | US; EU (Ireland) on Business+ | None documented | "EU residency options"; custom region Enterprise | EU primary DCs; no choice documented |

## Per-provider notes

### Knock

- **Feed.** `@knocklabs/react` provides `NotificationFeedPopover` and `NotificationIconButton`, plus the headless hooks `useKnockClient`, `useNotifications` and `useNotificationStore` under `KnockProvider` / `KnockFeedProvider`. In headless mode you must call `feedClient.markAllAsSeen()` yourself. Prebuilt CSS is themed with `--rnf-*` variables, and you can skip it entirely. ([overview](https://docs.knock.app/in-app-ui/react/overview), [headless](https://docs.knock.app/in-app-ui/react/headless/feed), [repo](https://github.com/knocklabs/javascript/tree/main/packages/react))
- **Web push.** There is no first-party web push. It goes through FCM, which means the Firebase SDK, `firebase-messaging-sw.js` and a VAPID key from Firebase. You run `Notification.requestPermission()` yourself, then call `knock.user.setChannelData`. Knock's OneSignal integration is mobile only. ([FCM tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm), [OneSignal](https://docs.knock.app/integrations/push/one-signal))
- **Email.** Native channel. Postmark is supported, with delivery and bounce tracking. ([email integrations](https://docs.knock.app/integrations/email/overview))
- **Node.** `@knocklabs/node`, written in TypeScript. Trigger with `client.workflows.trigger(key, { recipients, data })`. Its `jose` dependency is ESM-only, which is fine for Cadence because Cadence is ESM. ([knock-node](https://github.com/knocklabs/knock-node))
- **Auth.** "Enhanced security mode" is set per environment. When it is on, the server signs an RS256 JWT with `signUserToken(userId, { expiresInSeconds })` and passes it as `userToken`. The docs suggest leaving it off in development. A DevUI user switch fits: re-sign in the loader for the session user. ([security](https://docs.knock.app/in-app-ui/security-and-authentication), [KnockProvider](https://docs.knock.app/in-app-ui/react/sdk/components/knock-provider))
- **Environments.** Development and Production, plus custom environments, each with its own keys. You commit changes and promote them, with rollback. ([environments](https://docs.knock.app/concepts/environments))
- **Pricing.** Free covers 10k messages a month and keeps Knock branding. Starter is $250 a month for 50k messages, then $0.005 per message. A message is "a single message delivered to a single user on a single channel", so feed plus push counts twice. At our scale that is about $250–500 a month. ([pricing](https://knock.app/pricing))
- **Preferences.** PreferenceSets cover channel types, channels, workflows and categories, and there is a hosted preference page. In React there is no drop-in component, so you build the UI on the client or API. ([preferences](https://docs.knock.app/preferences/overview), [React prefs](https://docs.knock.app/in-app-ui/react/preferences))
- **Real-time.** WebSocket, via `listenForUpdates()`. ([client](https://github.com/knocklabs/javascript/tree/main/packages/client))
- **Region.** US only today. EU residency is "expected by the end of 2026". ([plans](https://docs.knock.app/manage-your-account/knock-plans))
- **SSR.** No docs guidance. Remix/Vite SSR once broke with an ESM import error ([issue #19](https://github.com/knocklabs/javascript/issues/19)), fixed in February 2024. Render the feed client-only.

### Novu Cloud

- **Feed.** `@novu/react` provides `<Inbox/>` plus the hooks `useNotifications`, `useCounts`, `usePreferences` and `useNovu`. Styling goes through `appearance` (`variables`, `elements`, `icons`), and `elements` accepts **Tailwind classes**. Styles are injected, with no CSS import. This is the best Tailwind fit of the six. ([inbox](https://docs.novu.co/platform/inbox/overview), [styling](https://docs.novu.co/platform/inbox/configuration/styling), [hooks](https://docs.novu.co/platform/sdks/react/hooks/use-notifications))
- **Web push.** Not native. It goes through FCM web push, OneSignal, Pushpad, Pusher Beams or a push webhook. You handle the permission prompt yourself. Pairing Novu with OneSignal as the web push provider is a documented route. ([push](https://docs.novu.co/platform/integrations/push), [FCM](https://docs.novu.co/platform/integrations/push/fcm))
- **Email.** Native channel. Postmark is supported, and a demo provider exists. ([email](https://docs.novu.co/platform/integrations/email))
- **Node.** `@novu/api` (TypeScript, ESM). Trigger with `novu.trigger({ workflowId, to: { subscriberId }, payload })`. ([TS SDK](https://docs.novu.co/platform/sdks/server/typescript))
- **Auth.** HMAC, toggled per environment. The server computes `sha256(secretKey, subscriberId)` and passes it as `subscriberHash`. The hash never expires, so it is weaker than a JWT with an expiry, but DevUI switching is trivial. ([prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production))
- **Environments.** Development and Production. Production is view-only, and you publish from Development. Custom environments need the Team plan. ([environments](https://docs.novu.co/platform/developer/environments))
- **Pricing.** Free covers 10k runs, 5 workflows and **24-hour feed retention**. Pro starts at $30 for 30k runs, then $1.20 per 1k, with 20 workflows and **7-day feed retention**. Team starts at $250 for 250k runs, with 90-day retention and up to 10 environments. A run is one trigger for one subscriber, counted once however many channels it uses. At our scale Pro costs $30–54 a month. If Cadence wants feed history older than a week, Team costs $250. ([pricing](https://novu.co/pricing), [billing](https://docs.novu.co/platform/account/billing))
- **Preferences.** Global per channel and per workflow, plus "critical" workflows users cannot opt out of. Preferences are built into the Inbox UI, and `usePreferences` supports a custom UI. ([preferences](https://docs.novu.co/platform/concepts/preferences))
- **Real-time.** WebSocket, on by default. ([hooks](https://docs.novu.co/platform/sdks/react/hooks/use-notifications))
- **Region.** US or EU (Frankfurt) on all plans, chosen when the account is created. You cannot migrate later. ([security](https://docs.novu.co/platform/additional-resources/security))
- **SSR.** The docs say "Inbox does not support server-side rendering" ([migration guide](https://docs.novu.co/platform/inbox/migration-guide)), and the Next.js quickstart uses `'use client'`. Render the feed client-only.

### Courier

- **Feed.** `@trycourier/courier-react` v9 wraps the web components in `@trycourier/courier-ui-inbox`, which render in **shadow DOM**, so Tailwind classes do not reach inside them. Theming uses `CourierInboxTheme` and custom slot components. For a Tailwind/Radix UI, go headless with `useCourier()`. ([inbox](https://www.courier.com/docs/platform/inbox/inbox-overview), [React SDK](https://www.courier.com/docs/sdk-libraries/courier-react-web), [courier-web](https://github.com/trycourier/courier-web))
- **Web push.** Not native. Pusher Beams is the only provider documented with a web mode. The FCM and OneSignal pages do not mention browser push. There are no permission helpers. ([push](https://www.courier.com/docs/integrations/push/overview), [Pusher Beams](https://www.courier.com/docs/integrations/push/pusher-beams))
- **Email.** Routed through your own ESP. Postmark is supported, with failover across providers. ([email](https://www.courier.com/docs/external-integrations/email/intro), [Postmark](https://www.courier.com/docs/external-integrations/email/postmark))
- **Node.** `@trycourier/courier` v10 (TypeScript, CJS and ESM). Send with `client.send.message({ message: { to, content } })`. ([courier-node](https://github.com/trycourier/courier-node))
- **Auth.** The server calls `POST /auth/issue-token` with scopes `user_id:<id> inbox:read:messages inbox:write:events` and an `expires_in`. The client calls `signIn({ userId, jwt })`, and calling it again switches users. ([authenticate users](https://www.courier.com/docs/in-app/authenticate-users))
- **Environments.** Test and Production, with separate keys, templates and logs. ([go live](https://www.courier.com/docs/guides/go-live))
- **Pricing.** Free covers 10k sends a month, and test sends count. Business is $0.005 per send, and a send covers all of its channels. At our scale that is about $100–250 a month. ([pricing](https://www.courier.com/pricing), [billing](https://www.courier.com/docs/workspaces/billing))
- **Preferences.** Per topic and per channel, with required topics. There is a hosted page and an embeddable `courier-ui-preferences` component. ([preference center](https://www.courier.com/docs/in-app/preference-center))
- **Real-time.** WebSocket at `wss://realtime.courier.io`. **Region.** US, or EU (Ireland) on Business and up. ([datacenters](https://www.courier.com/docs/workspaces/datacenters))
- **SSR.** The components are client-only. The React bundle has no `"use client"` directive and registers custom elements, so it needs a client-only wrapper or a lazy import.

### MagicBell

- **Feed.** `@magicbell/react` v1.14 provides FloatingInbox, Inbox, ToastContainer, UserPreferences and WebPushButton, styled with prebuilt CSS, CSS variables and render overrides. It documents no hooks. For headless use, combine `magicbell-js/user-client` and `magicbell-js/realtime`. `@magicbell/react-headless` is no longer in the monorepo and looks legacy. ([React](https://www.magicbell.com/docs/libraries/magicbell-react), [magicbell-js](https://www.magicbell.com/docs/libraries/magicbell-js))
- **Web push.** **Native.** You configure VAPID in MagicBell and host your own `/sw.js`, which calls `importScripts`. `WebPushButton` handles the permission flow. ([web push](https://www.magicbell.com/docs/channels/web_push))
- **Email.** SES, Mailgun, SendGrid, Ping and SMTP. **No Postmark integration.** SMTP could point at Postmark, but that route is not documented. ([email](https://www.magicbell.com/docs/channels/email), [SMTP](https://www.magicbell.com/docs/channels/email/smtp))
- **Node.** `magicbell-js` v1.8 (TypeScript, CJS and ESM), authenticated with a Project JWT. **Auth.** An HS256 user JWT signed with the project secret. ([user auth](https://www.magicbell.com/docs/api/authentication/user))
- **Environments.** None. The docs say to "create projects for staging/production", and Free allows 1 project. ([quick start](https://www.magicbell.com/docs/quick-start))
- **Pricing.** Free is **1k deliveries a month**. Startup is $249 for 50k deliveries, then $0.0025 each. A delivery is counted per channel, so at our scale that is about $249–374 a month. ([pricing](https://www.magicbell.com/pricing))
- **Preferences.** Per channel and per category, with a `UserPreferences` component. **Real-time.** WebSocket. **Region.** Not documented. **SSR.** The ESM build ships `"use client"`, which React Router ignores, so render client-only.

### SuprSend

- **Feed.** `@suprsend/react` provides the drop-in `Inbox`, `NotificationFeed` and `Preferences`. `@suprsend/react-core` provides `SuprSendFeedProvider`, `useFeedClient()` and `useFeedData()` for a fully headless Tailwind/Radix feed. The drop-in uses Emotion with theme objects and component overrides. `@suprsend/react-inbox` is end of support. ([React](https://docs.suprsend.com/docs/react-1.md), [headless feed](https://docs.suprsend.com/docs/react-sdk-headless-feed.md), [repo](https://github.com/suprsend/suprsend-react-sdk))
- **Web push.** **Native**, via VAPID (`vapidKey` from the dashboard). You host `serviceworker.js` at the site root, which `importScripts` `@suprsend/web-sdk`. `webpush.registerPush()` must be called from a user gesture, and `notificationPermission()` reports the current state. There is no prebuilt prompt UI. ([webpush](https://raw.githubusercontent.com/suprsend/suprsend-react-sdk/main/docs/webpush.md))
- **Email.** Routed through your ESP. **Postmark is supported**, including a delivery-status webhook. ([Postmark](https://docs.suprsend.com/docs/postmark.md))
- **Node.** `@suprsend/node-sdk` v1.15. Trigger with `workflows.trigger(new WorkflowTriggerRequest(body))`. It ships types but is **CommonJS only**, so Cadence's ESM server relies on Node's CJS interop. ([trigger](https://docs.suprsend.com/docs/trigger-workflow.md), [package.json](https://raw.githubusercontent.com/suprsend/suprsend-node-sdk/main/package.json))
- **Auth.** A public key plus an **ES256 JWT** signed by the server with a workspace signing key (`entity_id`, `exp`, `scope`). It is mandatory when enhanced security is on. A switch is passing a new `distinctId` and `userToken` to `SuprSendProvider`; `distinctId=null` logs out, and `refreshUserToken` handles expiry. This fits the DevUI cleanly. ([client auth](https://docs.suprsend.com/docs/client-authentication.md))
- **Environments.** Sandbox, Staging and Production workspaces, fully isolated with separate keys. Test mode is in Staging. ([workspaces](https://docs.suprsend.com/docs/workspaces.md))
- **Pricing.** Free covers 10k notifications a month, with SuprSend branding and **no preferences**. Essentials is $100 a month for 50k, then $2 per 1k, still without preferences. Business is $250 a month for 50k, then $5 per 1k, and adds category and channel preferences, a ready-made preference UI and a hosted preference page. An MTU (monthly tracked users) model is available on request. ([pricing](https://www.suprsend.com/pricing))
- **Preferences.** Category, channel-within-category and overall channel APIs. ([preferences](https://raw.githubusercontent.com/suprsend/suprsend-react-sdk/main/docs/preferences.md))
- **Real-time.** Socket, which disconnects after one minute hidden by default. **Region.** The docs say "We support EU-data residency options"; pricing lists a custom region as Enterprise only. ([security](https://docs.suprsend.com/docs/security))
- **SSR.** No guidance. The service worker must be served from `/` (Cadence's `public/`). Render the feed client-only.

### OneSignal: fails two hard requirements

- **No web feed.** In-app messages are mobile only and transient, and OneSignal "does not currently store the history of messages sent to each individual user". ([in-app](https://documentation.onesignal.com/docs/en/in-app-messages-setup), [activity feed](https://documentation.onesignal.com/docs/en/create-an-activity-feed))
- **No signed web auth.** JWT identity verification is "Not yet supported" on the Web SDK. Anyone with the app ID can `login(externalId)`. ([identity verification](https://documentation.onesignal.com/docs/en/identity-verification))
- **Strong at web push.** `react-onesignal` is native, with `OneSignalSDKWorker.js` self-hosted, automatic VAPID, slidedown and bell prompts, and `allowLocalhostAsSecureOrigin`. Web push is free up to 10k subscribers per send. ([React setup](https://documentation.onesignal.com/docs/en/react-js-setup), [permission requests](https://documentation.onesignal.com/docs/en/permission-requests), [pricing](https://onesignal.com/pricing))
- **Email.** No Postmark. ([email setup](https://documentation.onesignal.com/docs/en/email-setup))
- **Possible role.** OneSignal could serve as the **web push transport behind Novu**, which lists OneSignal as a push provider. Knock's and Courier's OneSignal integrations are not documented for web.

## Ranked shortlist

The final pick is left to the provider grilling ticket.

1. **SuprSend.** The only provider that meets every hard requirement natively, including first-party VAPID web push alongside a headless feed.
   - Fits Cadence: Postmark email, an ES256 signed JWT with clean user switching, Sandbox/Staging/Prod workspaces, and a 10k free tier.
   - Costs: $100 a month without preferences, or $250 with the preference centre.
   - Watch: a CJS-only Node SDK, no SSR docs, and EU residency that is vague below Enterprise.
2. **Novu Cloud.** The cheapest option and the best Tailwind fit.
   - Fits Cadence: `appearance` takes Tailwind classes, there are hooks, preferences are built into the Inbox, Postmark is supported, US/EU regions are on all plans, and a run counts once however many channels it uses.
   - Costs: Pro at $30–54 a month.
   - Watch: web push needs FCM or OneSignal, and feed retention is 24 hours on Free and 7 days on Pro. 90 days takes Team at $250. The HMAC subscriber hash never expires.
3. **Knock.** The most mature docs and model.
   - Fits Cadence: RS256 signed tokens, environments with promotion, a rich preferences model and Postmark.
   - Costs: $250–500 a month, the most expensive here, because messages are billed per channel.
   - Watch: web push via FCM only, US-only data until EU arrives, and no React preferences component.
4. **Courier.**
   - Fits Cadence: Postmark, Test/Prod environments, per-send billing at about $100–250, and a preference UI.
   - Watch: web push only via Pusher Beams, shadow-DOM components that ignore Tailwind (so headless `useCourier()` is required), and EU only on Business.

**Not shortlisted**

- **MagicBell.** No Postmark, a 1k free tier, no environments (separate projects instead), no documented region, and $249+ a month.
- **OneSignal.** No web feed and no signed web auth. It is a candidate only as a push transport behind Novu.

## Could not verify

- **Rendering behaviour (all providers).** No vendor documents React Router SSR. Assume client-only rendering of the feed for every provider. React 19 support is unconfirmed for SuprSend and MagicBell (no React peer range declared).
- **Knock.**
  - Whether in-app feed messages are billed exactly like other channels (the docs imply yes).
  - Whether the 10k free limit is shared across environments.
  - A `usePreferences` hook in `@knocklabs/react`.
- **Novu.**
  - Whether `@novu/node` is deprecated in favour of `@novu/api` (npm returned 403).
  - Per-category preferences.
  - Pro and Team member limits (the pricing table was partly garbled in fetch).
- **Courier.**
  - Whether FCM or OneSignal routes support browser push.
  - Whether inbox-only sends are priced differently.
  - Whether the $0.005 rate applies to the first 10k sends on Business. The pricing page was read that way, but it should be confirmed with sales.
- **MagicBell.**
  - Region.
  - Postmark over SMTP.
  - Whether VAPID keys are generated or supplied.
  - Free-tier overage.
- **SuprSend.**
  - Whether the drop-in `Preferences` component ships (the README and docs disagree).
  - Whether EU residency is available below Enterprise.
  - The socket transport.
  - Whether the JWT can be skipped in Staging.
- **OneSignal.**
  - The total cap on free web push subscribers (only a per-send cap is shown).
  - Region choice.
  - Push transport details.
- **Postmark in the codebase.** Cadence's current Postmark integration (`app/lib/mailer.server.ts`) is not on `main`, so nothing was checked against existing email code.
