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
**Impact:** Unauthenticated exposure of authenticated user data

---

## TL;DR

This wasn't the textbook web cache deception where you append `/x.css` and ride the extension. There was no file-extension trick at all.

The cache had a rule scoped to a **parent path** — meant to cache a harmless *public* sibling endpoint sitting under that path. But the same parent path also contained an **authenticated** endpoint returning per-user data. Because the rule was scoped to the directory prefix instead of the specific public route, the cache happily stored the authenticated responses too — **overriding the origin's `Cache-Control: private`**. Anyone could then replay the cached copy with no session and read another user's data.

---

## The usual WCD vs. this one

Classic web cache deception relies on a **path–extension confusion**: origin ignores a fake `/x.css` suffix and serves private content, cache sees `.css` and stores it.

This finding is a different species. The trigger wasn't an extension — it was **where the cache rule was anchored**. A rule written for:

```
/api/reviews/book/{id}          <- intended: public, cacheable
```

was actually scoped to the **prefix**:

```
/api/reviews/book/{id}/*        <- everything under here gets cached
```

So a *sibling* under the same parent that required authentication and returned private, per-user data got caught by the same rule. Two endpoints, one parent directory, one overly-broad cache rule. The public one was *supposed* to be cached. The private one came along for the ride.

---

## The finding

### The two endpoints under one roof

- **Public sibling** — cacheable by design, no auth, same response for everyone. This is what the cache rule was written for.
- **Authenticated endpoint** — same parent path, returned `[per-user data — fill in: profile / entitlements / etc.]`, and set `Cache-Control: private`.

Because the edge rule keyed on the shared parent prefix, it treated **both** as cacheable and ignored the `private` directive on the authenticated one.

### Reproduction

1. Authenticate as **Victim**. Request the authenticated endpoint under the shared parent path. Origin returns private data; edge stores it.

   ```
   GET /api/reviews/book/{id}/[authenticated-subpath] HTTP/1.1
   Host: [redacted]
   Cookie: [victim session]
   ```

   Response shows a cache **MISS** being stored despite `Cache-Control: private`:

   ```
   [Paste actual observed headers, e.g.:]
   Cache-Control: private
   CF-Cache-Status: MISS    -> HIT on replay
   Age: [n]
   ```

2. From a **clean, unauthenticated context** (incognito, no cookies, different IP), request the exact same URL.

3. Edge replays the cached response — **HIT** — serving Victim's private data to an attacker with no credentials.


---

## Impact

Any unauthenticated attacker who requests (or predicts) the cached URL reads authenticated user data — no session, no interaction required beyond the victim's response having been cached once. Depending on `[what the endpoint exposed]`, impact ranges from PII disclosure to `[account/session implications if applicable]`.



---

## Root cause

A single scoping mistake:

- The cache rule was anchored to a **parent path prefix** rather than the specific public route it was meant to accelerate.
- A **sibling authenticated endpoint** lived under that same prefix.
- The edge's prefix rule took precedence over the origin's `Cache-Control: private`, so private responses were stored and served to everyone.

The origin was doing the right thing (`private`). The cache overruled it because the rule was too broad.

---

## Remediation

- Scope cache rules to the **exact route** that is safe to cache, never a parent directory that may contain authenticated siblings.
- Make the edge **honor `Cache-Control: private` / `no-store` and `Set-Cookie`** — never cache a response carrying those, regardless of path rules.
- Add a guard: responses to requests with an `Authorization` header or session cookie should be non-cacheable by default.
- Audit existing prefix/wildcard cache rules for authenticated endpoints hiding under the same parent.

---

*Target and program intentionally undisclosed.*
