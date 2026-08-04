---
layout: post
title: "CVE-2026-34758: the test endpoints nobody put a guard on"
summary: "OneUptime applies its service authentication middleware to the notification send routes. The test routes, in the same files, got none, and one of them could buy phone numbers."
reading_time: "5 min"
tags: [cve, missing-authentication, nodejs, typescript]
lang: en
ref: oneuptime-unauth-notifications
cve: CVE-2026-34758
---

**CVE-2026-34758** · [GHSA-q253-6wcm-h8hp](https://github.com/advisories/GHSA-q253-6wcm-h8hp) ·
OneUptime · CWE-306 · Critical, CVSS 9.1 · published 2026-03-30 · fixed in 10.0.40

OneUptime is an open-source monitoring platform. It sends alerts, which means it holds Twilio
credentials, SMTP credentials and WhatsApp credentials for the organisation running it.

Its notification API has authentication. `ClusterKeyAuthorization.isAuthorizedServiceMiddleware`
is applied to `/send` and `/make-call`. Correctly, in every relevant file.

It was not applied to any of the `/test` endpoints, nor to phone number management.

```javascript
router.post(
  "/test",
  async (req, res, next) => {
    // no authentication middleware
    const toPhone = new Phone(body["toPhone"]);
    await WhatsAppService.sendWhatsApp(message, { ... });
```

These routes are exposed publicly through the `/notification` nginx route.

## What that gets you

Send SMS, place calls, send WhatsApp messages and send email, all on the victim
organisation's accounts and at their expense. The WhatsApp one needs a single parameter,
`toPhone`, and no configuration identifier at all.

The phone number management endpoints go further: an unauthenticated caller can **purchase**
phone numbers on the organisation's Twilio account.

That last one is why this is a critical and not a rate-limiting annoyance. The failure is not
that a stranger can make your server send a text. It is that a stranger can spend your money
and speak with your identity, to recipients they choose.

## The shape

"Test" and "preview" routes are a recurring blind spot, and the reason is psychological
rather than technical. Authentication gets attached to the routes that *do the thing*. A test
route feels like a diagnostic, something you press in your own settings page, so it does not
read as an action that needs protecting. But the code path underneath is the same code path:
it really does send the message, using the real credentials.

Whenever I map an API surface, the routes I check first are the ones whose names suggest they
are harmless: `/test`, `/verify`, `/preview`, `/check`, `/validate`, `/ping`. In this case the
guard was defined and correct thirty lines away, which made the gap obvious once the surface
was listed side by side.

## Impact and fix

Unauthenticated abuse of the victim's SMS, voice, WhatsApp and email channels, plus phone
number purchase. Financial cost, reputational cost, and a convincing phishing channel from a
number the recipients trust.

Fixed in 10.0.40 by applying the existing middleware to the test and phone number routes.

## Timeline

- Reported to the maintainers
- Fixed in 10.0.40
- Advisory published 2026-03-30, CVE-2026-34758 assigned
