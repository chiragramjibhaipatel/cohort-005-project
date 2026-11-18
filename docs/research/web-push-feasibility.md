# Is web push feasible for Cadence?

Research for [#4](https://github.com/chiragramjibhaipatel/cohort-005-project/issues/4) on the map [#2 Notification system spec](https://github.com/chiragramjibhaipatel/cohort-005-project/issues/2). Sources were checked in November 2026.

This note sets out facts and build cost. It does **not** decide whether Cadence uses web push; that is for [#7](https://github.com/chiragramjibhaipatel/cohort-005-project/issues/7) and [#8](https://github.com/chiragramjibhaipatel/cohort-005-project/issues/8).

## Short answer

- **Yes, technically.** Web push works with no provider account, no fees and only OSS code. The server signs requests with its own VAPID key pair and posts them to whichever push service the browser chose (Google FCM, Mozilla, Apple APNs, Microsoft WNS). Nobody has to sign up with any of them.
- **Desktop and Android:** it works in a normal browser tab in Chrome, Edge, Firefox, Opera and Samsung Internet, and in Safari 16+ on macOS 13+.
- **iPhone and iPad:** it works **only when the Student has added Cadence to the Home Screen** and opens it from there. A Safari tab on iOS cannot receive web push. Nothing in Safari 26.x or 27.x (latest is 27.1, 23 Oct 2026) changes this.
- **`npm run dev`:** this works without an account. `http://localhost` is a secure context, so subscribing and receiving work there in desktop browsers. Messages still go out through the real push services, so dev needs internet access. Testing on a real iPhone needs an HTTPS URL; a free TryCloudflare quick tunnel gives one without an account.
- **Build cost:** moderate. You need a service worker file, a web app manifest and icons, a subscribe/unsubscribe UI, one new table, one dependency (`web-push`), VAPID keys in env, and cleanup of dead subscriptions. Details are in "Build cost" below.

## 1. Browser support

From MDN's browser-compat-data (`api/PushManager.json`, `api/Notification.json`, fetched Nov 2026):

| Browser | `PushManager` since | Notes |
| --- | --- | --- |
| Chrome / Chrome Android | 42 | `subscribe()` requires `applicationServerKey` |
| Edge | 17 | `subscribe()` requires `applicationServerKey` |
| Firefox | 44 | From 72, `subscribe()` only works in response to a user gesture |
| Firefox Android | 48 | From 79, user gesture required |
| Opera, Samsung Internet | mirror Chrome | |
| Safari (macOS) | 16 | "Notifications are supported on macOS Ventura and later." |
| Safari iOS / iPadOS | 16.4 | "Notifications are supported in web apps saved to the home screen." `Notification` "is undefined, unless the page is a web app saved to the home screen." |
| Android WebView, iOS WKWebView | not supported | |

MDN calls the Push API Baseline "Widely available … since March 2023" ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)).

### iOS and iPadOS in detail

- **Home Screen only.** WebKit: "Now with iOS and iPadOS 16.4, we are adding support for Web Push to Home Screen web apps" ([WebKit, Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). Apple: "Add web push to Home Screen web apps in iOS 16.4 or later and Webpages in Safari 16 for macOS 13 or later" ([Apple, Sending web push notifications in web apps and browsers](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- **Manifest.** In 2023 a manifest with `display: standalone` or `fullscreen` was needed for the Home Screen icon to open as a web app ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). Since iOS/iPadOS 26, "By default, every website added to the Home Screen opens as a web app" and "there are now zero requirements for 'installability' in Safari" ([WebKit Features in Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)). No source I found says outright that push works for a Home Screen app that has no manifest. Ship a manifest anyway: it is cheap, and it supplies the name, icons and `id`.
- **Third-party browsers on iOS** (Chrome, Edge and others) "can now offer their users the ability to add websites and web apps to the Home Screen" ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). The result is still a WebKit Home Screen web app.
- **Permission** must be requested "in response to direct user interaction" ([WebKit 13878](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). Users manage it per web app in Settings › Notifications.
- **No silent pushes.** "Safari doesn't support invisible push notifications. … If you don't, Safari revokes the push notification permission for your site" ([Apple doc](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- **No Apple Developer account.** "You don't need to join the Apple Developer Program to send web push notifications" ([Apple doc](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- **EU.** Apple planned to remove Home Screen web apps in the EU under the DMA (iOS 17.4 betas), then reversed and kept them ([Apple, Update on apps distributed in the EU](https://developer.apple.com/support/dma-and-apps-in-the-eu/)). I found no later reversal.
- **Nothing new in 2026.** The Safari 26.2, 26.4, 27.0 (17 Sep 2026) and 27.1 (23 Oct 2026) release posts contain no web push or Home Screen push changes. 27.0 adds the Service Worker static routing API and some service worker and `Notification` bug fixes ([26.2](https://webkit.org/blog/17640/webkit-features-for-safari-26-2/), [26.4](https://webkit.org/blog/17862/webkit-features-for-safari-26-4/), [27.0](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/), [27.1](https://webkit.org/blog/18486/webkit-features-for-safari-27-1/)).

### Declarative Web Push (Safari only today)

- Safari on iOS/iPadOS 18.4+ and macOS 15.5+ can show a push **without a service worker**. The push payload is JSON with `"web_push": 8030` and a `notification` object that needs a `title` and a `navigate` URL. Subscriptions are also available from `window.pushManager` ([WebKit, Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)).
- It is now in the W3C Push API Working Draft of 16 Nov 2026: §3.3 "Declarative push message", and "Window includes PushManagerAttribute" ([W3C Push API](https://www.w3.org/TR/push-api/)).
- The format is backward compatible: "If it arrives to an older browser, it's handled imperatively by JavaScript as it always had been" ([WebKit 16535](https://webkit.org/blog/16535/meet-declarative-web-push/)).
- I found no primary source showing Chrome or Firefox shipping it. Cadence would still need a service worker for those browsers.
- **What it buys Cadence:** if the payload is always sent in the declarative format, Safari still shows the notification even when the service worker fails. Chrome and Firefox read the same JSON in the service worker. It is optional, and costs nothing extra on the server.

## 2. Service worker under React Router 8 (framework mode, Vite)

- The Push API needs "an active service worker" for imperative push, and "each subscription is unique to a service worker" ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)).
- **Simplest setup: a plain `public/sw.js`.** Vite serves files in `public/` "at root path `/` during dev, and copied to the root of the dist directory as-is" ([Vite, The public directory](https://vite.dev/guide/assets.html)). The file is served at `/sw.js`, so its default scope is `/`, which covers the whole app.
- **Production serving.** `react-router-serve` (8.3.0, `packages/react-router-serve/cli.ts`) serves `build/client` with plain `express.static(build.assetsBuildDirectory)`, with no `maxAge`. The service worker script is therefore not long-cached. Only `/assets/*` gets `immutable, maxAge: "1y"`.
- **Code style.** The file is not bundled or type-checked, so it is hand-written plain JS. Push handling is small: a `push` handler that calls `showNotification`, a `notificationclick` handler that focuses or opens a URL, and optionally `pushsubscriptionchange`.
- **Registration** happens in client code, for example a `useEffect` in `app/root.tsx` or in the settings page: `navigator.serviceWorker.register("/sw.js")`. React Router has no dedicated API for this. None is needed, because a framework-mode app is a normal Vite client bundle.
- **Alternative: `vite-plugin-pwa`.** Its `injectManifest` strategy compiles a TypeScript service worker and adds offline precaching ([vite-plugin-pwa injectManifest](https://vite-pwa-org.netlify.app/guide/inject-manifest)). I found no official React Router framework-mode recipe, and Cadence does not need offline caching for push. This is more machinery than the job needs.
- **Manifest.** Add `public/manifest.webmanifest` (`name`, `id`, `start_url`, `display: "standalone"`, 192 px and 512 px PNG icons) and `{ rel: "manifest", href: "/manifest.webmanifest" }` in `links` in `app/root.tsx`. Today Cadence only has `public/favicon.svg`.

## 3. VAPID key handling

- **What the key is.** `applicationServerKey` is "an ECDSA P-256 public key that the push server will use to authenticate your application server … all messages from your application server must use the VAPID authentication scheme" ([MDN PushManager.subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)). The JWT `exp` "MUST NOT be more than 24 hours" ahead. A subscription is bound to the public key it was created with, so **rotating the key orphans every existing subscription** and each one must be re-created ([RFC 8292](https://www.rfc-editor.org/rfc/rfc8292)).
- **No accounts.** "Chrome and Application Server Keys do not need any kind of project to be set up with Google or Firebase" ([web.dev push FAQ](https://web.dev/articles/push-notifications-faq)). Apple needs no developer account (see above).
- **Generating keys.** `npx web-push generate-vapid-keys --json` or `webpush.generateVAPIDKeys()`. "You should create these keys once, store them and use them for all future messages" ([web-push README](https://github.com/web-push-libs/web-push)).
- **Config.** In practice: `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` and `VAPID_SUBJECT` in env. The public key also has to reach the browser, through a loader or a resource route. For dev, generate a pair on first run if none is set, so `npm run dev` needs no setup.
- **Subject gotcha (Apple).** The VAPID subject must be a real `mailto:` or `https:` URI. "Apple Push may reject localhost-based subjects with `403 BadJwtToken`" ([web-push README](https://github.com/web-push-libs/web-push)). Apple lists `BadJwtToken` causes, including the subject not being a URL or `mailto:`, and `exp` more than a day ahead ([Apple doc](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)). Apple also says: "Don't refresh your JWT more frequently than once per hour."
- **Outbound network.** If outbound traffic is restricted, allow `https://*.push.apple.com` ([Apple doc](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)), plus FCM, Mozilla and WNS hosts.

### The `web-push` npm package

- `web-push@4.0.0`, published 6 Nov 2026, licence **MPL-2.0** (from `npm view web-push`). Node `>= 16`. Dependencies: `jws`, `asn1.js`, `http_ece`, `https-proxy-agent`.
- **4.0.0 is a big jump** from 3.6.7 (Jan 2024). It "migrate[s] package to ESM", moves to class syntax and changes the CLI ([v4.0.0 release](https://github.com/web-push-libs/web-push/releases/tag/v4.0.0)). It ships no TypeScript types. `@types/web-push` is still at 3.6.4, so Cadence would either pin `web-push@3.6.7` with `@types/web-push`, or use 4.x with a small local `.d.ts`.
- **`sendNotification` options** ([README](https://github.com/web-push-libs/web-push)):
  - `TTL`: default four weeks; Apple stores 30 days at most.
  - `urgency`: use `high` to deliver immediately.
  - `topic`: collapses notifications that share the same topic.
  - `contentEncoding`: `aes128gcm` by default.
  - `timeout`, `proxy`.
- **Errors.** A rejected promise exposes `statusCode`, `headers` and `body` from the push service.
- **Payload size.** Apple caps the payload at 4 KB ("`PayloadTooLarge`"). Keep the payload to title, body and URL.

## 4. How subscriptions expire or get revoked

- **`expirationTime`** is usually `null`. "In theory, this means the subscription never expires. In practice, however, browsers commonly let subscriptions expire" ([web.dev, Subscribing a user](https://web.dev/articles/push-notifications-subscribing-a-user)). I found no primary source giving Chrome's inactivity window.
- **The spec.** User agents "SHOULD attempt to refresh the push subscription before the subscription expires" and fire `pushsubscriptionchange`. `newSubscription` is `null` when no new one could be made. "When a permission is revoked … The user agent MUST deactivate the affected subscriptions" ([W3C Push API](https://www.w3.org/TR/push-api/)).
- **The server's signal is the send response.** `404` or `410` mean "the subscription has expired or is no longer valid … we need to remove the subscriptions from our database" ([web.dev, Sending messages with web push libraries](https://web.dev/articles/sending-messages-with-web-push-libraries)). Apple's table: `410` "The device token has expired", `413` payload too large, `429` too many requests, `403` auth error ([Apple doc](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- **Ways a subscription dies:**
  - The user blocks notifications or uninstalls the Home Screen app.
  - The user clears site data, or the browser expires the subscription.
  - The VAPID key is rotated.
  - Safari sees a push that showed no notification, and revokes permission.
- **Keeping it reliable:**
  1. Delete the row on 404/410.
  2. On each app load, compare `registration.pushManager.getSubscription()` with the server and upsert. This catches silent re-subscriptions.
  3. Optionally handle `pushsubscriptionchange` in the service worker. I did not find a primary source confirming Safari fires it, so do not rely on it alone.
- **The endpoint is a secret.** "The endpoint URL therefore needs to be kept secret, or other applications might be able to send push messages to your application" ([MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)). The subscribe and unsubscribe routes need the usual auth and CSRF care.

## 5. Testing locally over `http://localhost`

- **Desktop browsers.** `http://localhost` and `*.localhost` are secure contexts. Plain-http LAN IPs such as `http://192.168.x.x` are not ([MDN Secure contexts](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts)). So Chrome, Edge and Firefox on the dev machine can register the service worker, subscribe and receive push from `npm run dev` with no extra setup.
- **Internet access.** Delivery still goes Cadence → real push service → browser, so dev needs internet. It needs no account.
- **Browser must be running.** On desktop, "The only time a push won't be received is when the browser is completely closed" ([web.dev push FAQ](https://web.dev/articles/push-notifications-faq)).
- **Safari on macOS over `http://localhost`.** Apple does not document it either way. Forum reports are mixed, so verify it in the prototype or treat it as unknown.
- **iPhone or iPad.** The device must load Cadence over HTTPS with a real hostname, then Add to Home Screen. A LAN IP is not secure, and the phone's `localhost` is the phone itself. A free option with no account:
  - Command: `cloudflared tunnel --url http://localhost:5173`.
  - "You do not need a Cloudflare account or domain" ([TryCloudflare](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/)).
  - Limits: testing only, 200 in-flight requests, no SSE, and the hostname changes every run. The Home Screen app is tied to that origin, so it has to be re-added each time.
  - Vite blocks unknown hosts. Add the tunnel host via `server.allowedHosts` or `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` ([Vite server options](https://vite.dev/config/server-options.html)).
- **Automated tests** cannot drive a real push service. Unit-test the send and cleanup logic with `web-push` mocked (Vitest), as with any other outbound adapter.

## 6. Build cost

| Piece | What | Size |
| --- | --- | --- |
| Manifest + icons | `public/manifest.webmanifest`, 192/512 PNG icons, `apple-touch-icon`, `<link rel="manifest">` in `root.tsx` | small |
| Service worker | `public/sw.js`: `push` → `showNotification`, `notificationclick` → open or focus URL, optional `pushsubscriptionchange` | ~50 lines plain JS |
| Registration + subscribe UI | Register the service worker; an "Enable browser notifications" button that calls `subscribe({ userVisibleOnly: true, applicationServerKey })` inside the click handler; unsubscribe; feature detection | small–medium |
| iOS install guidance | When on iOS and `PushManager` is missing, show "Add Cadence to your Home Screen to get notifications" | small, but it is UX copy and design |
| Data | `push_subscriptions` table: user id, endpoint (unique), `p256dh`, `auth`, created/last-success timestamps, a device label. Drizzle migration and seed (see the `database-migrations` skill) | small |
| Routes | Resource route to upsert or delete a subscription for the current user; the public key exposed to the client | small |
| Sender | `web-push` dependency; VAPID config from env (auto-generated in dev); send to all of a user's subscriptions; delete on 404/410; log other failures | medium. Retries depend on the open "Delivery mechanics" question |
| Payload format | Optionally the declarative `{"web_push":8030,"notification":{title, body, navigate}}` shape, read by the service worker too | trivial |
| Dev capture | Undecided (map: "Development-time capture"). Real delivery works on localhost, but a DevUI or log view of sent pushes would make it visible without OS notification prompts | small–medium |
| Tests | Vitest for the sender (mocked `web-push`), the subscription route and the cleanup rule | small–medium |

**Running costs:** $0. There is no account, no quota fee and no SDK, and Cadence's own process sends the messages.

**Ongoing obligations:**
- Keep the VAPID private key secret and stable.
- Always show a notification for every push. Safari revokes permission otherwise.
- Prune dead subscriptions.

## Things that may affect other tickets

1. **iPhone reach is opt-in twice.** A buyer on iOS gets push only if they (a) add Cadence to the Home Screen and (b) tap "allow". Push cannot be the only surface for the buyer's approved/declined Notifications; something else (email or in-app) has to carry them. This matters for #7.
2. **Subscriptions belong to a browser, not a person.** One user can have many (laptop, phone), and one browser can hold a subscription that was created while another user was signed in. The DevUI user switcher does exactly this in dev. The data model and logout flow need to handle it: delete or re-assign on logout or user switch. This matters for the "Data model" and "Development-time capture" questions.
3. **Dev push leaves the machine.** `npm run dev` needs no account, but every real push goes through Google, Mozilla or Apple servers. A purely local capture mode, like a mail catcher, would need a switch that skips `web-push` in dev.
4. **`web-push` 4.0.0 shipped 6 Nov 2026** as ESM with no types. If #3 or #8 pick it, decide between 3.6.7 with `@types` and 4.x with local typings.
5. **Preferences.** The OS and browser hold their own per-site and per-app notification switch. A Cadence-side "web push on/off" preference sits on top of that, which matters for the preference-granularity question.

## Gaps (not confirmed by a primary source)

- Whether iOS 26+ delivers push to a Home Screen web app that has **no** manifest. Ship a manifest to sidestep this.
- Whether Safari fires `pushsubscriptionchange`.
- Whether Safari on macOS allows push subscriptions on `http://localhost`.
- Chrome's exact inactivity window before it expires a subscription.
- Whether Chrome or Firefox have shipped Declarative Web Push.
