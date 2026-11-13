# What web push takes for a browser-only app

Research for [#3](https://github.com/wharter1980/cohort-005-project-1/issues/3), part of the map [#1 Notifications in Cadence](https://github.com/wharter1980/cohort-005-project-1/issues/1). Researched 2026-11-13. The keep/drop decision belongs to [#6](https://github.com/wharter1980/cohort-005-project-1/issues/6). This note records the facts and the cost to Cadence.

Cadence today has no `manifest.webmanifest` and no service worker (`public/` holds only `favicon.svg` and `images/`). It is a React Router 7 framework-mode app with SSR (`react-router.config.ts`).

## 1. The moving parts

Web push is three standards working together: the **Push API**, the **Notifications API**, and **Service Workers** ([WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).

- **Secure context.** `PushManager` is `[SecureContext]`, and `subscribe()` rejects with `NotAllowedError` when the service worker scope is not `https` ([W3C Push API, Working Draft 12 Nov 2026](https://www.w3.org/TR/push-api/)). `localhost` is fine for development.
- **Service worker.** "For an app to receive push messages, it has to have an active service worker." The browser starts the worker when a push arrives and calls its `push` handler, which shows the message with `registration.showNotification()` ([MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)).
- **Subscription.** `pushManager.subscribe({ userVisibleOnly, applicationServerKey })` returns a `PushSubscription`, which holds an `endpoint` URL and the encryption keys (`p256dh`, `auth`). The server stores it per user and device. MDN says the endpoint is a secret capability URL, so the subscribe flow needs CSRF protection ([MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)).
- **VAPID keys.** `applicationServerKey` must be a P-256 public key, and it must differ from the message-encryption key ([W3C Push API](https://www.w3.org/TR/push-api/)). The server signs each push request with an ES256 JWT. The `exp` claim can be at most 24 h out, and `sub` should be a `mailto:` or `https:` contact. A push service rejects requests to a restricted subscription that are not signed by the matching private key. **If you rotate keys, every user has to subscribe again** ([RFC 8292](https://datatracker.ietf.org/doc/html/rfc8292)). MDN notes that Chrome and Edge require the key and `userVisibleOnly: true` ([MDN: PushManager.subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)).
- **Visible-only.** Chrome and Safari accept only `userVisibleOnly: true`, so every push must show a notification. WebKit: "The Web Push API is not an invitation for silent background runtime… Violations of the `userVisibleOnly` promise will result in a push subscription being revoked" ([WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/)).
- **Push service.** Each browser picks its own push service: FCM for Chrome, Mozilla's service for Firefox, APNs for Safari. The developer cannot choose it, but the Web Push protocol is the same for all of them ([web.dev: Push notifications overview](https://web.dev/articles/push-notifications-overview)). Payloads are encrypted, so the push service cannot read them. A push service must accept at least 4096 bytes of payload and returns 413 for anything larger ([web.dev: Web Push protocol](https://web.dev/articles/push-notifications-web-push-protocol)).

## 2. Permission prompt: UX, timing, and browser restrictions

- **A user gesture is required.** Firefox has rejected permission requests made outside a user gesture since version 72, and other browsers are moving the same way ([MDN: PushManager.subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)). Safari requires "an explicit user gesture" ([WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/)). Chrome and Firefox no longer allow the request from cross-origin iframes ([MDN: Notification.requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static)).
- **Never prompt on page load.** web.dev: "The worst thing you can do is to show the permission dialog to users as soon as they land on your site." Ask "at a time when the benefit is obvious." If no such moment exists, put a toggle in settings instead ([web.dev: Permission UX](https://web.dev/articles/push-notifications-permissions-ux)).
- **Use a double-permission (pre-prompt) pattern.** Show an in-app explainer first, and call the real browser prompt only when the user says yes. A "no" on your own dialog costs nothing ([web.dev: Permission UX](https://web.dev/articles/push-notifications-permissions-ux)).
- **A denial is effectively permanent.** After a block, the site generally cannot ask again. The user has to dig through browser settings to undo it ([web.dev: Permission UX](https://web.dev/articles/push-notifications-permissions-ux)). `requestPermission()` resolves to `granted`, `denied`, or `default`, and `default` should be treated as denied ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static)).
- **Chrome demotes prompts.** Since Chrome 98, a permission request shows only an animated chip by the address bar unless the user clicks it, and the chip collapses to a "blocked" icon. Chrome measured that about 85% of notification prompts were dismissed or ignored. It calls auto-requesting without a gesture a "bad practice" and warns that repeated requests risk auto-blocking ([Chrome for Developers: Permissions chip](https://developer.chrome.com/blog/permissions-chip)). Chrome also has a "quieter permission UI" for notifications, introduced January 2020 ([Chromium blog](https://blog.chromium.org/2020/01/introducing-quieter-permission-ui-for.html)); the post's body could not be retrieved, so its trigger criteria are not recorded here.

For Cadence, the natural prompt points are tied to events, for example after a Student submits a Refund Request ("Get notified when it's decided?"). The settings/preferences page holds the permanent toggle.

## 3. Platform support

| Platform | Support | Conditions |
| --- | --- | --- |
| Desktop Chrome / Edge / Firefox | Yes | Standard Push API. Push API `subscribe` has been Baseline "widely available… since March 2023" ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)). |
| Desktop Safari (macOS) | Yes | Safari 16 on macOS Ventura (13) and later. **No Apple Developer Program membership needed** ([WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/)). |
| Android Chrome / Firefox | Yes | Works in a normal browser tab. No install needed. |
| iOS / iPadOS | **Only for Home Screen web apps**, iOS/iPadOS 16.4+ | The user has to add the site to the Home Screen first, through Share → Add to Home Screen. A plain Safari tab cannot subscribe. The site **must serve a web app manifest with `display: standalone` or `fullscreen`**, or the site is saved as a bookmark and cannot use push. Permission must come from a user tap inside the installed app. Delivery goes through APNs, with no developer account needed. If you restrict egress, allow `*.push.apple.com`. Badging (`setAppBadge`) also works ([WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/); [MDN: manifest display](https://developer.mozilla.org/en-US/docs/Web/Manifest/display)). |

In practice, an iOS user reaches push only after two deliberate steps: installing to the Home Screen, then granting permission. Knock's own FCM tutorial code checks for iPhone and refuses to subscribe unless the app is installed, telling the user to "Add to Home Screen" ([Knock tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm)). Expect very low iOS reach unless Cadence actively promotes installing to the Home Screen.

## 4. Reliability limits

- **No delivery guarantee or receipt.** The push service holds a message while the device is offline, until the message's TTL expires. TTL 0 means "deliver now or drop." Push services may shorten the TTL. The protocol gives no confirmation that a notification was displayed ([web.dev: overview](https://web.dev/articles/push-notifications-overview); [web.dev: protocol](https://web.dev/articles/push-notifications-web-push-protocol)).
- **Subscriptions go stale.** A push service returns **404 or 410** for an expired or unsubscribed endpoint, and the server must delete it. A **429** response means rate-limited, and the server should honour `Retry-After` ([web.dev: protocol](https://web.dev/articles/push-notifications-web-push-protocol)). Subscriptions can have an `expirationTime`. The browser fires `pushsubscriptionchange` when it refreshes or invalidates one, and the worker must re-send the new subscription to the server ([W3C Push API](https://www.w3.org/TR/push-api/)).
- **One subscription per browser profile and device.** A user with a laptop and a phone has two subscriptions. Clearing site data or uninstalling the iOS Home Screen app removes the subscription.
- **Quotas vary and are not standardised.** "There is currently no standard mechanism." Firefox applies a quota to pushes that show no notification, which is irrelevant when every push is visible. Chrome has no limit ([MDN: Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)). `Urgency` and `Topic` headers let a server mark low priority or collapse superseded messages ([web.dev: protocol](https://web.dev/articles/push-notifications-web-push-protocol)).
- **OS-level suppression.** Focus modes, OS notification settings, and (on desktop) the browser not running can all delay or hide notifications. The specs give apps no signal when this happens.

Conclusion: web push suits time-sensitive nudges that **duplicate** a Notification already in the in-app feed and/or email. It should never be the only Channel for a mandatory Notification.

## 5. Vendor support

| Vendor | Web push support | How |
| --- | --- | --- |
| **Knock** | **No first-party web push channel.** "Knock does not have a first-party web push channel." | Uses FCM as a push channel. Cadence would create a Firebase project, take the VAPID key from Firebase, add the Firebase JS SDK and `firebase-messaging-sw.js`, then save the FCM token to the Knock user as channel data. Knock does not validate tokens, so pruning expired ones is Cadence's job ([Knock tutorial](https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm); [Knock push overview](https://docs.knock.app/integrations/push/overview)). |
| **Novu** | **Indirect.** No native VAPID channel found. | Push providers are FCM, Expo, APNs, OneSignal, Push Webhook, Pusher Beams, and Pushpad ([Novu push](https://docs.novu.co/platform/integrations/push)). FCM docs: "Desktop notifications for websites can be sent using FCM webpush," with tokens stored as subscriber `deviceTokens` ([Novu FCM](https://docs.novu.co/platform/integrations/push/fcm)). Pushpad and OneSignal are dedicated web push services. Push Webhook POSTs to your own endpoint, which could send via the `web-push` library, but Novu's docs do not describe that ([Novu Push Webhook](https://docs.novu.co/platform/integrations/push/push-webhook)). |
| **Courier** | **Indirect.** No first-party Web Push/VAPID provider found. | Its 12 push providers are APNs, FCM, Expo, OneSignal, Airship, Amazon SNS, Pusher Beams, Pusher, MagicBell, Pushbullet, Beamer, and NowPush. None is documented as browser push on Courier's own pages ([Courier push overview](https://www.courier.com/docs/integrations/push/overview.md)). Courier's FCM page covers only Android and iOS ([Courier FCM](https://www.courier.com/docs/integrations/push/firebase-fcm)). Browser push would go through a second vendor such as OneSignal or MagicBell, which means a second account and SDK. |
| **SuprSend** | **Yes, native.** | A WebPush vendor in the dashboard supplies the VAPID key. The React SDK's `SuprSendProvider` takes `vapidKey`, Cadence hosts `/serviceworker.js` (which loads `@suprsend/web-sdk`), and `suprSendClient.webpush.registerPush()` handles the prompt, subscription, and token sync. The docs advise calling it "on user action like button click… don't call this on page load" ([SuprSend WebPush](https://docs.suprsend.com/docs/webpush)). |
| **MagicBell** | **Yes, native.** | A Web Push channel is configured with your own VAPID `public_key`/`private_key` (dashboard or `PUT /v2/integrations/web_push`). Tokens are the raw `PushSubscription` (`endpoint`, `p256dh`, `auth`), with APIs to save, list, and delete them ([MagicBell web push channel](https://www.magicbell.com/docs/channels/web_push)). The `@magicbell/webpush` SDK registers the service worker (default `/sw.js`) and subscribes ([MagicBell WebPush SDK](https://magicbell.com/docs/libraries/webpush)). |

None of these vendors removes the platform constraints in sections 2–4. A vendor only handles sending, fan-out, and storing tokens. The iOS install requirement, the prompt UX, and the visible-only rule stay with Cadence.

## 6. What adopting web push would cost Cadence

**Work in the app, whichever vendor is chosen:**

1. A service worker served from the site root (for example `public/sw.js`, which Vite serves at `/sw.js`), registered on the client. It handles `push` (always `showNotification`), `notificationclick` (focus or open the deep link), and `pushsubscriptionchange`. Vendors that bring their own worker file (SuprSend, Firebase/Knock) still need that file hosted at a root path.
2. A web app manifest (`display: standalone`, name, icons including 192/512 PNGs, `start_url`) linked from `root.tsx`. This is mandatory for iOS. It is optional elsewhere but turns Cadence into an installable PWA.
3. Permission UX: a pre-prompt component shown at event-tied moments, a toggle on the preferences page, a "denied" state that explains how to unblock in browser settings, and on iOS an "Add to Home Screen" explainer for Safari-tab visitors.
4. Subscription and token lifecycle: sync to the vendor (or a `push_subscriptions` table if Cadence sends directly), one row per device, and pruning on 404/410.
5. Testing: push needs HTTPS or localhost and real browsers. It cannot be unit-tested the way the DevUI user switching can. Each persona needs manual QA on Chrome, Firefox, Safari macOS, Android, and an installed iOS app.

**Vendor-dependent cost:**

- **SuprSend or MagicBell:** native web push. VAPID keys are entered in the vendor dashboard, and the web push Channel takes part in the vendor's per-Channel preferences like email does. This is the lowest extra cost.
- **Knock or Novu:** a Firebase project as well (another account, config env vars, Firebase JS SDK of about tens of KB on the client), or for Novu possibly Pushpad or OneSignal. Firebase on Safari still goes through standard web push, with the same iOS constraints.
- **Courier:** a second push vendor (OneSignal, MagicBell, and so on) behind Courier, which conflicts with the "one orchestration vendor" preference in spirit.

**Practical reach:**

- **Desktop (all major browsers) and Android:** reachable once the user opts in. Opt-in is low by default, since Chrome reports about 85% of prompts dismissed or ignored. Context-driven prompting raises it.
- **iOS:** effectively only users who install Cadence to the Home Screen. For a course platform used mostly on laptops this may matter little, but iPhone-only Students should be treated as unreachable by push.
- **Reliability:** best effort with no delivery receipt, so web push can only supplement the in-app feed and email, never replace them for mandatory Notifications.

## Sources

- W3C Push API (WD 12 Nov 2026): https://www.w3.org/TR/push-api/
- RFC 8292 VAPID: https://datatracker.ietf.org/doc/html/rfc8292
- MDN Push API: https://developer.mozilla.org/en-US/docs/Web/API/Push_API
- MDN PushManager.subscribe: https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe
- MDN Notification.requestPermission: https://developer.mozilla.org/en-US/docs/Web/API/Notification/requestPermission_static
- MDN manifest `display`: https://developer.mozilla.org/en-US/docs/Web/Manifest/display
- WebKit, Meet Web Push: https://webkit.org/blog/12945/meet-web-push/
- WebKit, Web Push for Web Apps on iOS and iPadOS: https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/
- web.dev Push overview: https://web.dev/articles/push-notifications-overview
- web.dev Web Push protocol: https://web.dev/articles/push-notifications-web-push-protocol
- web.dev Permission UX: https://web.dev/articles/push-notifications-permissions-ux
- Chrome permissions chip: https://developer.chrome.com/blog/permissions-chip
- Knock: https://docs.knock.app/integrations/push/overview, https://docs.knock.app/tutorials/sending-web-push-notifications-with-fcm
- Novu: https://docs.novu.co/platform/integrations/push, https://docs.novu.co/platform/integrations/push/fcm, https://docs.novu.co/platform/integrations/push/push-webhook
- Courier: https://www.courier.com/docs/integrations/push/overview.md, https://www.courier.com/docs/integrations/push/firebase-fcm
- SuprSend: https://docs.suprsend.com/docs/webpush
- MagicBell: https://www.magicbell.com/docs/channels/web_push, https://magicbell.com/docs/libraries/webpush
