# Notification libraries and providers for Cadence

Researched 2026-11-18. Primary sources only. Anything not confirmed from a primary source is marked **unverified**.

## Question

Which libraries and providers could deliver Cadence's Notifications on **in-app**, **email** and **web push**, while staying free or open-source? The stack they must fit is React Router 8 (framework mode, Vite, Node server via `@react-router/serve`) and a single SQLite file via Drizzle (`better-sqlite3`). The app has no job runner, Redis, Postgres or Docker today.

## TL;DR

- **None of the three hosted platforms (Novu, Knock, Courier) sends browser Web Push (VAPID) natively.** Each routes "push" through FCM, OneSignal, Pusher Beams, Pushpad or similar. Knock says outright: "Knock does not have a first-party web push channel" ([Knock][knock-webpush]). Novu's Push Webhook provider lets you plug in your own sender ([Novu push webhook][novu-push-webhook]). SuprSend gives you VAPID keys in its dashboard, but its free tier has no preferences at all ([SuprSend pricing][suprsend-pricing]).
- **None of the hosted platforms sends email itself.** Knock, Courier and Novu (outside a Cloud-only demo provider) all need an email provider such as Resend, SES or SMTP behind them ([Knock email][knock-email], [Courier email][courier-email], [Novu demo][novu-demo]). Cadence needs an email provider whichever option is chosen.
- **Every hosted option needs an account or API key at dev time, and dev traffic counts toward quota.** Knock: "usage may be accrued in any Knock environment" ([Knock usage][knock-usage]). Courier counts sends from every environment, Test included ([Courier billing][courier-billing]). Novu's keyless Inbox expires after 24h ([Novu inbox setup][novu-inbox-setup]), and its demo email provider is Cloud-only ([Novu demo][novu-demo]). To meet the "`npm run dev` needs no provider account" rule, any hosted option needs a local adapter or stub behind an interface.
- **Self-hosted Novu (MIT core) is heavy next to a single-SQLite app.** It needs API, worker, WebSocket and dashboard services, plus MongoDB, Redis with AOF and S3-compatible storage. The minimum is a 4 vCPU / 8 GB VM ([Novu self-hosting][novu-selfhost]).
- **The plain-library route covers all three surfaces with no account at dev time.** That route is Nodemailer or Resend for email, `web-push` (MPL-2.0) for push, react-email (MIT) for templates and Drizzle tables for the inbox. The cost: Cadence builds the inbox, preference storage, unsubscribe tokens, the RFC 8058 endpoint and retries.
- **One-click unsubscribe is not legally required for Cadence's mail as Gmail defines it.** Gmail exempts transactional messages, and its bulk rules start at about 5,000 messages a day ([Gmail FAQ][gmail-faq]). The project's own constraint still requires an unsubscribe link. Of the hosted providers, Knock adds unsubscribe headers only on workflows marked "commercial" ([Knock commercial][knock-commercial]). Courier says "SendGrid is the only provider Courier emits these headers on today" ([Courier hosted page][courier-hosted]).

## Constraints recap (from the spec map)

1. Free or open-source only (self-hosted or free tier).
2. `npm run dev` must not need a provider account.
3. Production delivery is real; dev uses local capture.
4. Notifications are sent immediately, with no digests.
5. Recipients can opt out, including through an unsubscribe link in email.
6. First use case, at low volume:
   - Refund Request raised: notify Platform Admins.
   - Request approved or declined: notify the buyer.
   - Failed Refund: notify Platform Admins.

## Comparison table

| Option | In-app inbox | Email | Web Push (VAPID) | Free tier / license | `npm run dev` needs account or service? | Preferences / unsubscribe | Rough fit |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **Novu Cloud** | Yes (`@novu/react` Inbox) | Via your provider (Resend, SES, SMTP…); demo provider: 300/mo, only to your own login address | **No native.** FCM, OneSignal, Pushpad, Pusher Beams, Expo, APNS, or Push Webhook (DIY) | 10,000 workflow runs/mo, 5 workflows, 24h activity retention, 3 members, Novu branding | Yes: account + keys. Keyless Inbox expires after 24h | Global and per-workflow channel prefs, Inbox preferences UI. **Email unsubscribe link / List-Unsubscribe not documented** | Strained by constraint 2; push still DIY |
| **Novu self-hosted (Community)** | Yes | Via your provider (demo providers don't work self-hosted) | Same as Cloud | MIT core; `enterprise/` folder under a commercial license | Yes: Docker images for API, worker, ws and dashboard, plus MongoDB, Redis (AOF) and S3 for attachments | Same as Cloud; no email activity tracking; Novu branding can't be removed | Strained: needs infrastructure the app doesn't have |
| **Knock (free)** | Yes (feed + websocket, `@knocklabs/react`) | Via your provider; "Knock (testing only)" provider exists | **No.** "Knock does not have a first-party web push channel"; uses FCM | 10,000 messages/mo (each user × channel is one message); 30-day logs; Knock branding | Yes: account + API key. Dev usage counts; in-app can't be sandboxed | PreferenceSets, signed hosted preference page, one-click unsubscribe **for commercial workflows only** | Strained by constraint 2; push via FCM |
| **Courier (Developer)** | Yes (Courier Inbox, `@trycourier/courier-react`) | Via your provider (Resend, SMTP, SES…) | **No native.** FCM, OneSignal, Pusher Beams, MagicBell… | 10,000 sends/mo (one template to one recipient = one send, across channels). Inbox, preference and retention limits not stated | Yes: account + key. Test-env sends count | Hosted preference page, `{$.urls.unsubscribe}`; one-click headers **SendGrid only** | Strained by constraint 2; push via FCM |
| **SuprSend (free)** | Yes (with SuprSend branding) | Via vendor (unverified which) | VAPID key from SuprSend dashboard; SDK service worker loaded from CDN. Delivery path not documented | 10,000 notifications/mo; **no preferences on free** | Yes: account + key | Preferences and hosted page are **paid only** | Ruled out on free tier by constraint 5 |
| **Plain libraries** (Nodemailer or Resend + `web-push` + react-email + Drizzle) | Build it (SQLite table + loader/UI) | Nodemailer (MIT-0) SMTP to any provider, or Resend API (free 3,000/mo, 100/day) | **Yes, native.** `web-push` (MPL-2.0) signs VAPID and encrypts payloads | All OSS; Resend has a free tier | **No.** Nodemailer `jsonTransport`/`streamTransport` or Mailpit for email; VAPID keys generated locally | Build it: preference table, signed unsubscribe URL, RFC 8058 POST route, headers via `list`/`headers` | Fits all constraints; the most to build |

## Per-option details

### Novu Cloud (free tier)

- **Surfaces:**
  - Channels on Free: Email, In-app, SMS, Chat and Push ([Novu pricing][novu-pricing]).
  - In-app uses the Novu Inbox (`@novu/react`, React 18 or 19) ([Novu inbox setup][novu-inbox-setup]). The npm package is ISC-licensed, version 3.20.0 ([npm registry][npm-novu-react]).
  - **Web push:** the supported push providers are FCM, Expo, APNS, OneSignal, Push Webhook, Pusher Beams and Pushpad, and the page never mentions VAPID ([Novu push][novu-push]). There is no native VAPID sender. Push Webhook POSTs the payload (signed with HMAC `x-novu-signature`) to a URL you control, and "Your endpoint delivers the push" ([Novu push webhook][novu-push-webhook]). That endpoint could call `web-push` itself.
- **Free-tier limits:** $0, "No credit card required". 10,000 workflow runs a month, up to 5 workflows, 2 environments, 24h activity feed retention, up to 3 team members, Novu branding kept. Subscribers appear unlimited, but the comparison table rendered poorly, so treat that one as low-confidence. Pro starts at $30/mo ([Novu pricing][novu-pricing]).
- **Dev story:**
  - The Inbox needs an `applicationIdentifier` from a Novu account. Keyless mode is "only for testing", and its data "expires in 24h" ([Novu inbox setup][novu-inbox-setup]).
  - The demo email provider allows 300 per month and only sends "to the address associated with your logged-in Novu account". It is "only available in Novu cloud", not self-hosted or local ([Novu demo][novu-demo]).
  - `npx novu dev` lists `NOVU_SECRET_KEY` as a prerequisite ([Novu studio/CLI][novu-studio]).
  - **Verdict:** `npm run dev` would need an account unless the Novu calls sit behind a local adapter.
- **Preferences:**
  - Three layers: workflow channel defaults, subscriber global preferences and subscriber per-workflow preferences. "the most specific layer that defines a channel wins".
  - Critical workflows are hidden from subscriber preferences.
  - Subscribers edit preferences in the Inbox Preferences view or through the API ([Novu preferences][novu-prefs]).
  - **Email unsubscribe:** neither the preferences docs nor a site search turned up a documented email unsubscribe link, `List-Unsubscribe` header or hosted preference page. Treat this as **not supported out of the box (unverified absence)**. Cadence would build the link and endpoint and call the preferences API.
- **What Novu owns:** in-app message storage and real-time delivery, preference storage and evaluation, workflow and template editing, and the activity log. Retry behaviour toward downstream providers was not verified.
- **What Cadence builds:** the email provider account, the push sender (via Push Webhook) or an FCM project, the unsubscribe link and endpoint, and the dev adapter.

### Novu self-hosted (Community edition)

- **License:** "the core technology is fully open source, licensed under MIT license". The `enterprise` folder and its subfolders are under an enterprise license. The repo ships `LICENSE-MIT`, `LICENSE-ENTERPRISE` and `EE-PACKAGES-LICENSE` ([Novu GitHub][novu-gh]).
- **What it needs to run:**
  - Single VM: 4 vCPU / 8 GB RAM for all services, MongoDB (2 GB RAM, 20 GB disk), Redis (2 GB, "AOF enabled on the queue cluster to prevent job loss") and at least 10 GB of S3-compatible storage ([Novu self-hosting][novu-selfhost]).
  - The Docker deployment runs the `api`, `worker`, `ws` and `dashboard` images plus MongoDB and Redis.
  - Required secrets: `JWT_SECRET`, `STORE_ENCRYPTION_KEY` (32 characters), `NOVU_SECRET_KEY` and `HOST_NAME`.
  - S3 is only needed for email attachments. The compose file "does not start a LocalStack container" ([Novu docker][novu-docker]).
- **Feature gaps versus Cloud:**
  - No Snooze, and Novu branding can't be removed in Inbox or email.
  - No email activity tracking.
  - Activity feed retention is "Infrastructure dependent".
  - The Inbox, Bell and user preferences components are all available ([Novu self-hosted vs cloud][novu-diff]).
  - Social login is not supported ([Novu self-hosting][novu-selfhost]).
  - Demo email and SMS providers don't work self-hosted ([Novu demo][novu-demo]).
- **Fit:** No provider account is needed. But `npm run dev` would depend on a separately running Docker stack (MongoDB + Redis + four services), and production would need the same. The app runs none of that infrastructure today.

### Knock (free tier)

- **Surfaces:**
  - In-app feed with "websocket support, no infrastructure setup required" and React components ([Knock in-app][knock-inapp]). `@knocklabs/react` is MIT, version 0.15.1 ([npm registry][npm-knock]).
  - Email is sent through a provider you connect: Amazon SES, Cloudflare Email, MailerSend, Mailgun, Mailjet, Mailtrap, Mandrill, Postmark, Resend, SendGrid, SMTP, SparkPost, plus "Knock (testing only)" ([Knock email][knock-email]).
  - **Web push:** "Knock does not have a first-party web push channel." The tutorial uses FCM ([Knock web push tutorial][knock-webpush]).
- **Free-tier limits:**
  - $0, up to 10,000 messages a month.
  - Unlimited workflows, channels, team members and environments; configurable feed retention.
  - Preferences and in-app feed included; Knock branding kept; 30-day log retention.
  - Starter costs $250/mo ([Knock pricing][knock-pricing]).
  - **Metering:** "A message is counted each time a notification is successfully delivered to a single user on a single channel", so email plus in-app to one user counts as two ([Knock usage][knock-usage]).
- **Dev story:**
  - "Regardless of plan, usage may be accrued in any Knock environment."
  - Sandbox mode stops delivery and sandboxed messages don't count, but "The in-app channel doesn't support sandbox mode" ([Knock usage][knock-usage]).
  - An account and API key are required. No local or offline mode is documented ([Knock in-app][knock-inapp]).
- **Preferences and unsubscribe:**
  - Each user has a `PreferenceSet` of opt-outs by channel, category or workflow.
  - Knock "hosts and renders a preference page for each user, accessible via a signed URL", for the default set only ([Knock preferences][knock-prefs]).
  - One-click unsubscribe works only for workflows or broadcasts marked **commercial**: Knock "Adds required unsubscribe headers to all emails sent through that workflow or broadcast". The docs don't name the exact headers ([Knock commercial][knock-commercial]).
  - Cadence's refund notifications are transactional. Their email opt-out would rely on the hosted preference page link, not one-click headers, unless Cadence marks them commercial (semantically wrong) or builds its own.
- **What Knock owns:** feed storage, real-time delivery, preference storage and evaluation, the hosted preference page, templates and workflows, and logs.
- **What Cadence builds:** the email provider account, an FCM project for push, and a dev adapter.

### Courier (Developer / free tier)

- **Surfaces:**
  - Courier Inbox with real-time sync, read and archive state, and React packages `@trycourier/courier-react` v9 ([Courier in-app][courier-inapp]). The package is MIT, version 9.3.5 ([npm registry][npm-courier]).
  - Email: "Courier isn't an email provider. It sends through the providers you connect". Options include SES, SendGrid, Postmark, Mailgun, Resend, SMTP and others ([Courier email][courier-email]).
  - **Web push:** the push providers are APNs, FCM, Expo, OneSignal, Airship, SNS, Pusher Beams, Pusher, MagicBell, Pushbullet, Beamer and NowPush ([Courier push][courier-push]). None of them is a native VAPID sender.
- **Free-tier limits:**
  - "10,000 sends per month". Business costs $0.005 per send ([Courier pricing][courier-pricing]).
  - "One send is one template sent to one recipient". A multi-channel template counts once, and sends from every environment, including Test, count ([Courier billing][courier-billing]).
  - **Not stated on the pricing or billing pages:** user, Inbox, retention or branding limits on the free plan.
- **Dev story:** an account and API key are required, and test sends count. No local mode was found.
- **Preferences and unsubscribe:**
  - Hosted Preference Center at `view.notificationcenter.app`, linked through `{$.urls.preferences}`.
  - `{$.urls.unsubscribe}` gives a one-click opt-out from the message's subscription topic. It is empty if the template has no topic and does nothing on required topics ([Courier hosted page][courier-hosted]).
  - With "Unsubscribe Headers" turned on for a topic, Courier adds `List-Unsubscribe` and `List-Unsubscribe-Post`. But "SendGrid is the only provider Courier emits these headers on today". For other providers you set the headers through a provider override ([Courier hosted page][courier-hosted]).
  - Whether the hosted page is on the free plan is **not stated**.
- **What Courier owns:** inbox storage, preferences, the hosted page, templates and routing.
- **What Cadence builds:** the email provider account, an FCM or OneSignal setup for push, and a dev adapter.

### SuprSend (free tier), briefly

- **Free tier:** "10k Notifications/mo" with all channels, including web push and in-app. The in-app feed carries SuprSend branding. **Channel preferences, category preferences, the preference center and the hosted preference page are not included on Free.** Preferences arrive with the $250/mo Business plan ([SuprSend pricing][suprsend-pricing]).
- **Web push:** you pass a `vapidKey` taken from "SuprSendDashboard --> Vendors --> WebPush". The service worker loads SuprSend's SDK from jsDelivr ([SuprSend webpush][suprsend-webpush]). This looks like native VAPID, but the delivery path isn't documented. **Unverified.**
- **Fit:** Ruled out on the free tier by constraint 5, because opt-out needs preferences.

### Plain-library route

| Piece | Package | License / version (npm, 2026-11-18) | Notes |
| --- | --- | --- | --- |
| Email transport | `nodemailer` | MIT-0, 10.6.0 ([npm][npm-nodemailer]) | SMTP to any provider. `list.unsubscribe` sets `List-Unsubscribe` ([Nodemailer list headers][nm-list]). `List-Unsubscribe-Post` must go through `headers` ([Nodemailer custom headers][nm-headers]). Note: the repo's LICENSE file reads as plain MIT ([LICENSE][nm-license]) while npm metadata says MIT-0. |
| Email API (alternative) | `resend` | MIT, 6.38.1 ([npm][npm-resend]) | Free: 3,000 emails/mo, **100/day**, 3 domains, 30-day retention ([Resend pricing][resend-pricing]). Needs a verified domain with DKIM and SPF records ([Resend add domain][resend-domain]). Custom `List-Unsubscribe` and `List-Unsubscribe-Post` go in `headers`. "Resend doesn't manage contact lists for transactional emails" ([Resend unsubscribe][resend-unsub]). |
| Templates | `react-email` / `@react-email/render` | MIT, 6.11.16 / 2.3.1 ([npm][npm-react-email]) | Async `render()` with a `plainText` option ([react-email render][re-render]). "converted into HTML and sent using any email service provider", Nodemailer included ([react-email GitHub][re-gh]). |
| Web push | `web-push` | **MPL-2.0**, **4.0.0 published 2026-11-06**, `node >= 16` ([package.json][wp-pkg], [npm][npm-webpush]) | `generateVAPIDKeys()`, `sendNotification()` with TTL, urgency and topic; encryption handled for you. On failure the error carries `statusCode` ([web-push README][wp-gh]). CLI: `web-push generate-vapid-keys`. |
| In-app inbox | Drizzle + SQLite | n/a | A table of notifications per recipient with read state, rendered by a loader. Real-time needs polling or SSE (not researched). |

- **Surfaces:** all three. Web push talks straight to the browser vendor's push service with VAPID, so no FCM project or account is needed (see the cross-cutting notes).
- **Free / OSS:** fully OSS libraries. The only paid-tier exposure is the production email provider: Resend free, or any SMTP relay.
- **Dev story, with no account and no extra service:**
  - Email: Nodemailer `jsonTransport` or `streamTransport` builds the message without delivering it ([Nodemailer stream][nm-stream]), or point SMTP at Mailpit.
  - Push: VAPID keys are generated locally, and service workers register on `http://localhost` because it counts as a secure context ([MDN secure contexts][mdn-secure]).
  - In-app: plain SQLite.
- **Preferences and unsubscribe:** all of it is Cadence's to build:
  - A preferences table.
  - A signed, unauthenticated unsubscribe URL in the email body and in `List-Unsubscribe`.
  - An HTTPS POST route that meets RFC 8058: no redirect, and no reliance on cookies.
  - A DKIM signature covering both headers ([RFC 8058][rfc8058]). The production provider supplies the DKIM signature.
- **What Cadence builds:** everything, meaning inbox storage, preference storage, unsubscribe, push subscription storage (prune expired subscriptions, which RFC 8030 signals with **404** ([RFC 8030][rfc8030])), and retries. The app has no job runner, so retries would be in-process attempts or a "pending/failed" status column re-driven by a later trigger. That design is not researched here.

## Cross-cutting notes

### Web Push on Apple platforms

- **iOS / iPadOS:**
  - Needs **iOS or iPadOS 16.4 or later**, and only for **Home Screen web apps**. "A web app that has been added to the Home Screen can request permission."
  - The manifest `display` must be `standalone` or `fullscreen`, and permission must be requested "in response to direct user interaction".
  - It uses the standard Push API, Notifications API and Service Workers. "You do not need to be a member of the Apple Developer Program." Allowlist `*.push.apple.com` if you filter outbound traffic ([WebKit iOS 16.4][webkit-ios]).
  - **Consequence:** web push to iPhone buyers needs Cadence to ship a web app manifest and an "Add to Home Screen" flow.
- **macOS:** Web Push "supported in Safari 16 on macOS Ventura", with no Apple Developer Program needed ([WebKit Meet Web Push][webkit-meet]).
- **Unverified:** Apple's own developer page wouldn't render, so I couldn't confirm from Apple that Safari uses VAPID specifically. The WebKit posts only say "standards-based".

### Web Push protocol facts

- VAPID is RFC 8292 ([RFC 8292][rfc8292]). The push protocol is RFC 8030.
- The application server "MUST include the TTL" header.
- Sending to an expired subscription returns **404**. 410 is for undelivered messages when a receipt was requested ([RFC 8030][rfc8030]). In practice, many libraries and guides treat both 404 and 410 as "delete subscription" (**unverified** convention).
- Web push in dev still needs internet access, because the subscription endpoint is the browser vendor's push service. **Unverified:** MDN's Push API page didn't state this. I'm inferring it from the endpoint being a "capability URL" ([MDN Push API][mdn-push]).

### Email unsubscribe and deliverability

- **RFC 8058 one-click** ([RFC 8058][rfc8058]):
  - `List-Unsubscribe` "MUST contain one HTTPS URI".
  - `List-Unsubscribe-Post` "MUST contain the single key/value pair "List-Unsubscribe=One-Click"".
  - The POST "MUST NOT include cookies, HTTP authorization, or any other context information".
  - The sender "MUST NOT return an HTTPS redirect".
  - Both headers "MUST be covered by the [DKIM] signature".
- **Gmail:**
  - Senders of 5,000+ messages a day: "Marketing messages and subscribed messages must support one-click unsubscribe", plus a visible link in the body ([Gmail sender guidelines][gmail-guidelines]).
  - The FAQ says "Transactional messages are excluded from this requirement", with password resets and reservation confirmations as examples.
  - The bulk threshold is about 5,000 a day to personal Gmail accounts, and once met, the sender stays bulk ([Gmail FAQ][gmail-faq]).
  - All senders need SPF or DKIM, PTR records, TLS, and a spam rate under 0.3% ([Gmail sender guidelines][gmail-guidelines]).
- **Yahoo:**
  - Bulk senders need "a functioning list-unsubscribe header, which supports one-click unsubscribe for marketing and subscribed messages". POST under RFC 8058 is "highly recommended", and mailto is acceptable.
  - There must be a visible body link, and requests must be honoured within two days.
  - The page doesn't explicitly exempt transactional mail ([Yahoo sender best practices][yahoo]).
- **Implication for Cadence:** refund notifications are transactional and low volume, so mailbox-provider rules don't force one-click headers. The project constraint (an unsubscribe link in email) is stricter than what the mailbox providers require. Adding RFC 8058 headers anyway is cheap on the plain-library route.

### Dev-time local capture

- **Nodemailer `jsonTransport` / `streamTransport`:** in-process with zero services. Produces JSON or raw RFC 822 output without delivering ([Nodemailer stream][nm-stream]). Simplest fit for "no separate service".
- **Mailpit:** MIT, single static binary (also Homebrew and Docker), SMTP on 1025 and web UI on 8025. Has a REST API for integration tests, "List-Unsubscribe syntax validation" and an HTML client-compatibility check ([Mailpit][mailpit]). It is a separately running process, but not an account.
- **Ethereal:** a fake SMTP service with "no signup or email verification required". Accounts can come from `nodemailer.createTestAccount()`, "messages never get delivered", and the UI shows the last 15 days ([Ethereal][ethereal]). It is an external network service, so it is less in the spirit of local capture.
- **Resend:** no local mode found. The SDK needs an API key, so dev would swap the transport.
- **Hosted platforms:** none offers an offline or local mode (see each section). The Novu demo provider is Cloud-only and sends only to your own address. Knock sandbox mode doesn't cover in-app.

## Open questions for later tickets

1. Does any surface need to be real-time (websocket or SSE) in-app, or does refresh-on-navigation work for low-volume refund notifications?
2. Is web push to iPhone users in scope? It requires a manifest, a Home Screen install flow and a user-gesture prompt.
3. Is the production email sender Resend (free: 100 a day) or generic SMTP? Which domain carries the DKIM signature?
4. Should Cadence add RFC 8058 headers to transactional mail even though Gmail doesn't require them?
5. Without a job runner, what is the retry policy for a failed email or push? Options include synchronous attempts, a status column with re-drive, or an admin-visible failure.
6. If a hosted platform is chosen, what is the shape of the local adapter that keeps `npm run dev` account-free, and does it fork behaviour enough to hide bugs?
7. Platform Admins are a group. Fan-out to "all Platform Admins" means N recipients, which counts N times toward Knock and Courier quotas. That is irrelevant at low volume, but worth noting.
8. Novu's email unsubscribe support: confirm with Novu whether any unsubscribe link or header mechanism exists. The docs don't document one.

## Sources

[novu-pricing]: https://novu.co/pricing
[novu-selfhost]: https://docs.novu.co/community/self-hosting-novu/overview
[novu-docker]: https://docs.novu.co/community/self-hosting-novu/deploy-with-docker
[novu-gh]: https://github.com/novuhq/novu
[novu-push]: https://docs.novu.co/platform/integrations/push
[novu-push-webhook]: https://docs.novu.co/platform/integrations/push/push-webhook
[novu-prefs]: https://docs.novu.co/platform/concepts/preferences
[novu-inbox-setup]: https://docs.novu.co/platform/inbox/setup-inbox
[novu-demo]: https://docs.novu.co/platform/integrations/demo-providers
[novu-studio]: https://docs.novu.co/framework/studio
[novu-diff]: https://docs.novu.co/community/self-hosted-and-novu-cloud
[knock-pricing]: https://knock.app/pricing
[knock-usage]: https://docs.knock.app/manage-your-account/managing-usage
[knock-webpush]: https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
[knock-email]: https://docs.knock.app/integrations/email/overview
[knock-inapp]: https://docs.knock.app/in-app-ui/overview
[knock-prefs]: https://docs.knock.app/preferences/overview
[knock-commercial]: https://docs.knock.app/preferences/commercial-unsubscribe
[courier-pricing]: https://www.courier.com/pricing
[courier-billing]: https://www.courier.com/docs/workspaces/billing
[courier-email]: https://www.courier.com/docs/external-integrations/email/intro
[courier-push]: https://www.courier.com/docs/external-integrations/push/intro-to-push.md
[courier-inapp]: https://www.courier.com/docs/in-app/overview.md
[courier-hosted]: https://www.courier.com/docs/platform/preferences/hosted-page
[suprsend-pricing]: https://www.suprsend.com/pricing
[suprsend-webpush]: https://docs.suprsend.com/docs/webpush
[resend-pricing]: https://resend.com/pricing
[resend-unsub]: https://resend.com/docs/dashboard/emails/add-unsubscribe-to-transactional-emails
[resend-domain]: https://resend.com/docs/add-a-domain
[nm-stream]: https://nodemailer.com/transports/stream
[nm-list]: https://nodemailer.com/message/list-headers
[nm-headers]: https://nodemailer.com/message/custom-headers
[nm-license]: https://github.com/nodemailer/nodemailer/blob/master/LICENSE
[re-gh]: https://github.com/resend/react-email
[re-render]: https://react.email/docs/utilities/render
[wp-gh]: https://github.com/web-push-libs/web-push
[wp-pkg]: https://github.com/web-push-libs/web-push/blob/master/package.json
[npm-webpush]: https://www.npmjs.com/package/web-push
[npm-nodemailer]: https://www.npmjs.com/package/nodemailer
[npm-resend]: https://www.npmjs.com/package/resend
[npm-react-email]: https://www.npmjs.com/package/react-email
[npm-novu-react]: https://www.npmjs.com/package/@novu/react
[npm-knock]: https://www.npmjs.com/package/@knocklabs/react
[npm-courier]: https://www.npmjs.com/package/@trycourier/courier-react
[mailpit]: https://github.com/axllent/mailpit
[ethereal]: https://ethereal.email/
[gmail-guidelines]: https://support.google.com/a/answer/81126
[gmail-faq]: https://support.google.com/a/answer/14229414
[yahoo]: https://senders.yahooinc.com/best-practices/
[rfc8058]: https://www.rfc-editor.org/rfc/rfc8058.html
[rfc8030]: https://www.rfc-editor.org/rfc/rfc8030.html
[rfc8292]: https://datatracker.ietf.org/doc/rfc8292
[webkit-ios]: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
[webkit-meet]: https://webkit.org/blog/12945/meet-web-push/
[mdn-secure]: https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts
[mdn-push]: https://developer.mozilla.org/en-US/docs/Web/API/Push_API

- Novu: [pricing](https://novu.co/pricing), [self-hosting overview](https://docs.novu.co/community/self-hosting-novu/overview), [Docker deploy](https://docs.novu.co/community/self-hosting-novu/deploy-with-docker), [GitHub/license](https://github.com/novuhq/novu), [push providers](https://docs.novu.co/platform/integrations/push), [Push Webhook](https://docs.novu.co/platform/integrations/push/push-webhook), [preferences](https://docs.novu.co/platform/concepts/preferences), [Inbox setup](https://docs.novu.co/platform/inbox/setup-inbox), [demo providers](https://docs.novu.co/platform/integrations/demo-providers), [Studio](https://docs.novu.co/framework/studio), [self-hosted vs cloud](https://docs.novu.co/community/self-hosted-and-novu-cloud)
- Knock: [pricing](https://knock.app/pricing), [managing usage](https://docs.knock.app/manage-your-account/managing-usage), [web push via FCM](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm), [email providers](https://docs.knock.app/integrations/email/overview), [in-app](https://docs.knock.app/in-app-ui/overview), [preferences](https://docs.knock.app/preferences/overview), [commercial unsubscribe](https://docs.knock.app/preferences/commercial-unsubscribe)
- Courier: [pricing](https://www.courier.com/pricing), [billing](https://www.courier.com/docs/workspaces/billing), [email](https://www.courier.com/docs/external-integrations/email/intro), [push](https://www.courier.com/docs/external-integrations/push/intro-to-push.md), [in-app](https://www.courier.com/docs/in-app/overview.md), [hosted preference page](https://www.courier.com/docs/platform/preferences/hosted-page), [preferences editor](https://www.courier.com/docs/platform/preferences/preferences-editor)
- SuprSend: [pricing](https://www.suprsend.com/pricing), [web push](https://docs.suprsend.com/docs/webpush)
- Libraries: [web-push](https://github.com/web-push-libs/web-push), [Nodemailer stream/json](https://nodemailer.com/transports/stream), [Nodemailer list headers](https://nodemailer.com/message/list-headers), [Nodemailer custom headers](https://nodemailer.com/message/custom-headers), [react-email](https://github.com/resend/react-email), [react-email render](https://react.email/docs/utilities/render), [Resend pricing](https://resend.com/pricing), [Resend unsubscribe](https://resend.com/docs/dashboard/emails/add-unsubscribe-to-transactional-emails), [Resend add domain](https://resend.com/docs/add-a-domain), npm registry (`registry.npmjs.org`, fetched 2026-11-18) for versions and licenses
- Dev capture: [Mailpit](https://github.com/axllent/mailpit), [Ethereal](https://ethereal.email/)
- Standards and mailbox providers: [RFC 8058](https://www.rfc-editor.org/rfc/rfc8058.html), [RFC 8030](https://www.rfc-editor.org/rfc/rfc8030.html), [RFC 8292](https://datatracker.ietf.org/doc/rfc8292), [Gmail sender guidelines](https://support.google.com/a/answer/81126), [Gmail sender FAQ](https://support.google.com/a/answer/14229414), [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/), [WebKit: iOS 16.4 web push](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/), [WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/), [MDN secure contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts), [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
