---
title: "Unauthenticated DB Access and Account Takeover via a Leaked Staging Domain"
date: 2026-05-09
draft: false
tags: ["improper-access-control", "js-reversing", "recon", "account-takeover"]
categories: ["writeup"]
summary: "A hostname referenced inside a production JavaScript bundle pointed at a staging environment where every /api/admin/* route was reachable without authentication, including one that returned a password reset link for any user ID."
ShowToc: true
TocOpen: false
---

## TL;DR

The production application had a string inside one of its public JavaScript bundles pointing at a staging environment. That staging environment sat on a completely different apex domain, ran the same application, and had every `/api/admin/*` route open to unauthenticated requests. Five of those routes accepted arbitrary database queries against Redis, the main relational store, the message store, audit logs, and analytics. A sixth, `/api/admin/liaUrl`, returned a password reset link for any user ID you handed it. With sequential public-facing user IDs, that endpoint alone was a path to taking over staff and developer accounts.

## Target & Scope

- **Asset (production)**: `*.redacted.com`
- **Asset (staging)**: a separate apex on a different TLD, technically out of scope, accepted on a one-time exception
- **Severity**: High (CVSS 8.6)
- **Weakness**: Improper Access Control (CWE-284)

## Recon

I had been mapping the production app for a while when I started pulling its JavaScript bundles into a local folder so I could grep through them at leisure. Bundles are noisy, but they leak constantly: hostnames, undocumented routes, feature flag names, references to internal tools. The bundle that mattered here was at:

```
https://www.redacted.com/REDACTED/bundle.js
```

Buried inside that file was a string referencing a hostname I had not seen anywhere else. It was not a subdomain of the production apex. It was not on any of the documented related domains. It was an entirely separate registrable apex on a different TLD, with a name that had no obvious connection to production. Nothing in the public footprint pointed at it. No cert transparency search under the predictable queries would have returned it. The only reason it was discoverable at all was that someone had baked the URL into a bundle the public internet could pull.

I loaded it in a browser. Same application as production with a noticeably staging feel: a slightly different theme, missing assets, older copy in a few places. Different user database too; my production credentials were rejected. So I made a fresh test account.

## The admin endpoints

Same application meant the same routes, in theory. On production, anything under `/api/admin/*` returned 401 with no body. On staging I started walking the same paths to see which ones were registered.

```
GET /api/admin/redisquery
GET /api/admin/query
GET /api/admin/messageDbQuery
GET /api/admin/serverAuditLogDbQuery
GET /api/admin/analyticsDbQuery
```

All five returned 200 to an unauthenticated request. No session cookie, no auth header, no IP allowlist check, nothing. The names tell you what each one is: a Redis query interface, a SQL query interface against the main database, a query interface for the message store, one for server audit logs, one for analytics.

I want to be specific about what I actually executed here. Once I confirmed each endpoint accepted input and returned structured data, I stopped. Running real queries against live data, even on a staging environment, is the kind of thing that turns a bug bounty submission into an incident. Confirming the auth check was missing was enough to write the report.

## The password reset endpoint

The endpoint that pushed the severity from "this is bad" to "this is account takeover" was this one:

```
GET /api/admin/liaUrl?numericalUserId=<USER_ID>
```

`liaUrl` reads as "log in as URL", the kind of helper customer support staff use to impersonate accounts during debugging. The handler took a numeric user ID and returned a password reset link tied to that user. No auth required.

User IDs in this application were sequential integers and visible to anyone using the site. Staff and developer accounts sat in a predictable low range. The full path from "no account" to "any account, including staff" was three steps:

1. Pick a target user ID, or just iterate.
2. Hit `/api/admin/liaUrl?numericalUserId=<id>` on staging.
3. Follow the returned password reset link and set a new password.

That is account takeover of any user the attacker chose to point it at.

## Reporting and retest

I sent the report flagging that the staging hostname was technically outside the documented scope, and argued that the URL was reachable from a string baked into a production bundle. Triage accepted it on a one-time basis given the severity, which is the right call. A scope policy is meant to focus researchers, not give an asset cover when the asset is one curl away from a production page.

The first patch did not hold. On retest I could still hit every endpoint. I noted the bypass, the team rolled a second patch within hours, and on the next retest every route returned `Forbidden - not authorized`. They added an extra $50 on top of the bounty for the rework, which was a nice touch.

## Takeaways

A few things worth pulling out:

1. JavaScript bundles are the cheapest recon you have access to. You are looking for hostnames, undocumented routes, feature flag names, references to internal tools. None of this requires anything fancier than `curl | grep`. Scope policies do not list assets the program does not know it is leaking.

2. Staging environments need the same auth posture as production. The "nobody knows the URL so it does not matter" model stops being true the moment a developer hardcodes that URL into something the public internet can pull, which is constantly.

3. When you find a leaked asset like this, frame it correctly in the report. The point is not "I tested an out-of-scope thing." The point is "this hostname was disclosed from a production asset, here is the exposure on it, here is the file from production that surfaced it." Triage teams generally accept these even when the asset is technically outside scope, because the alternative is treating a public leak as untestable.

4. Sequential public integer IDs are not a vulnerability on their own. They are an amplifier. Anything that takes a user ID and returns something sensitive becomes O(n) trivial when the IDs are guessable.

The fix here was the right one. Gate every admin route the same way regardless of which environment is serving it. The scope-of-trust for an admin endpoint should not depend on which DNS name you reached it through.
