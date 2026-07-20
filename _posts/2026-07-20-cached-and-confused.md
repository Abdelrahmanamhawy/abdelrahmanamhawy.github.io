---
title: "Cached and Confused — A Story of Cache Deception"
excerpt: "A private API turned public — not by the classic extension trick, but by a cache rule scoped to the wrong parent path."
date: 2026-07-20
header:
  image: /assets/images/cached-and-confused-cover.png
  teaser: /assets/images/cached-and-confused-cover.png
toc: true
toc_sticky: true
---


# Cached and Confused
*How a cache rule scoped to a parent directory turned a private API into a public one*

**Type:** Web Cache Deception via cache-rule misconfiguration (CWE-524: Information Exposure Through Caching)
**Impact:** Unauthenticated exposure of users' private reviews

---

## TL;DR

This wasn't the textbook web cache deception where you append `/x.css` and ride the extension. There was no file-extension trick at all.

A CDN cache rule was anchored to a **parent path** so it could accelerate a harmless, public endpoint sitting under it — the published reviews for a title. But that same parent path also held an **authenticated** endpoint that returned the requesting user's **private reviews**. Because the rule matched the whole prefix rather than just the public resource, the CDN cached the private response too — overriding the origin's `Cache-Control: private` — and then served one user's private reviews to anyone who asked.

---

## Not the usual WCD

Classic web cache deception relies on **path–extension confusion**: the origin ignores a fake `/profile/x.css` suffix and serves private content, while the cache sees `.css` and stores it as a static asset.

None of that happened here. No suffix, no extension, no path-normalization trick. The bug lived entirely in **where the cache rule was anchored** — a scoping mistake in the CDN config, not a parser disagreement.

That distinction is the whole point. An extension-based WCD scanner walks straight past this. So does a developer who "checked for cache deception" by testing `.css` and `.js` suffixes and moved on. Nothing was malformed. The rule was doing exactly what it was told — it was just told to cover too much.

---

## The setup

For any given title, the reviews API lived under one parent path:

```
/api/reviews/book/{id}/…
```

Under that parent sat two very different resources:

- **Public reviews** — the published reviews for that title. Identical for every visitor, no authentication, completely safe to cache. This is the resource the cache rule was actually written for.
- **Private reviews** — the requesting user's own reviews that were not public: content they had authored but deliberately kept to themselves. Authenticated, per-user, and returned with `Cache-Control: private`.

Two resources, one parent, one cache rule that couldn't tell them apart.

---

## The bug

The CDN rule was anchored to the parent prefix (`/api/reviews/book/{id}/`) so the public reviews would be served fast from the edge. But a prefix rule matches **everything** beneath it. The private-reviews response — despite carrying `Cache-Control: private` — matched that same prefix, so the edge stored it and treated it as cacheable.

And this is the part that turns a config quirk into a vulnerability: many CDNs let a path-based cache rule **override** the origin's cache headers. The origin said "never store this, it's private." The edge said "this path is cacheable." The edge won.

### What that looks like in practice

1. A victim, authenticated, requests their private reviews. The origin returns them with `Cache-Control: private`. The edge, following its prefix rule, writes the response to cache (a cache **MISS** that gets stored).
2. An attacker — no cookies, no session, a different network entirely — requests the exact same URL.
3. The edge replays the stored copy: a cache **HIT**. The victim's private reviews are handed to an unauthenticated stranger.

The origin never even sees step 2. From its side, nothing suspicious happened — the leak occurs entirely at the edge, on infrastructure the application team often doesn't think of as part of their auth boundary.

---

## Why this is worse than it sounds

"Private reviews leaked" can read as low severity until you sit with what a private review actually is.

It's **content a user deliberately chose not to publish.** That choice is the entire expectation being violated — drafts, candid opinions, notes they weren't ready to attach their name to. The privacy wasn't incidental; it was the point.

It also **ties an identity to specific interests and specific words.** Learning which titles a given user privately reviewed, and what they wrote about them, is exactly the sort of profile that powers targeted phishing and social engineering. It's personal in a way a leaked config value never is.

And crucially, **a cache serves everyone.** This isn't a one-victim IDOR you have to exploit per target. Once a private response lands in the cache, every subsequent request for that URL receives it until the TTL expires. With cache keys built from predictable identifiers, an attacker doesn't wait for leaks — they walk the identifier space and farm private content across users at scale.

So the blast radius isn't "one object disclosed." It's "any private review that happens to be requested gets rebroadcast to the open internet for the lifetime of the cache entry."

---

## Root cause

One scoping mistake, three compounding conditions:

- The cache rule was anchored to a **parent path prefix** instead of the specific public resource it was meant to accelerate.
- A **private, authenticated resource** lived under that same prefix.
- The edge's prefix rule **took precedence over the origin's `Cache-Control: private`**, so private responses were stored and replayed.

Every component behaved "correctly" in isolation. The origin set the right header. The CDN honored its rule. The vulnerability lived in the seam between them — which is exactly where this class of bug always hides.

---

## Remediation

- Scope cache rules to the **exact resource** that is safe to cache — never a parent path that can contain authenticated siblings.
- Make the edge **respect `Cache-Control: private` / `no-store` and `Set-Cookie`**: a response carrying any of those must never be cached, regardless of path rules.
- Treat any request bearing a session cookie or `Authorization` header as non-cacheable by default.
- Audit existing prefix and wildcard cache rules specifically for private endpoints sitting beneath them.

---

*Target and program intentionally undisclosed.*
