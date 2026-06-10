---
layout: post
title:  "DNS Negative Caching"
date:   2026-06-10 02:27:13 +0000
categories: infrastructure networking dns
---

# It Wasn't the Code: How DNS Negative Caching Broke Every Build

There's a particular kind of bug that erodes your confidence faster than any other: the one where the same code passes, then fails, then fails again — and every fix you apply makes absolutely no difference. I had one of those recently. The culprit turned out to be something no amount of staring at the build logs would have revealed: DNS negative caching. Here's how it played out, and why retrying only made things worse.

## The symptom

Our build pipeline provisions ephemeral environments on AWS. Each deployment creates a handful of new Route53 DNS records — per-instance hostnames for the services that come up — and a later step in the build resolves one of those names to carry on.

For a while, everything was fine. Then builds started failing. Not occasionally — consistently, on every run. The maddening part was that the build logic hadn't changed, and I'd watched earlier runs succeed. So I did what we all do: I started fixing things. Tweak, run, fail. Tweak, run, fail. None of it landed, because — as it turned out — the fault was never in the code at all.

## The red herring

My first instinct was DNS propagation. New record, not yet visible, classic. And the moment you go looking, you find the line that strikes fear into anyone running an automated pipeline: DNS changes can take up to 24 hours to propagate.

That figure sent me into a brief panic, and it's worth saying clearly why it shouldn't have. The "up to 24 (or 48) hours" wording applies to **delegation changes** — standing up a brand-new hosted zone, or changing the authoritative name servers at the registrar. For a record created inside a zone that's already delegated and live, that's simply not the situation. Route53 propagates record changes to its authoritative name servers within roughly sixty seconds. Propagation wasn't my problem.

## The real culprit: a cached "no"

The actual issue was **negative caching** — and once I understood it, the on/off/on/off pattern made perfect sense.

When a resolver asks for a name that doesn't exist yet, it gets an NXDOMAIN — "no such name" — back. Crucially, resolvers cache that negative answer, just as they cache positive ones. How long they hold it is governed by the zone's SOA record: the negative-cache window is the lesser of the SOA record's own TTL and the minimum field buried at the end of it. On a default Route53 zone, that works out to 900 seconds — fifteen minutes.

Now string the sequence together:

1. The build creates a new record.
2. A downstream step resolves that hostname *before* the record has finished propagating — a window of only a minute or so, but enough.
3. The resolver gets an NXDOMAIN and caches it.
4. Sixty seconds later the record is live and perfectly resolvable on the authoritative servers — but the resolver is still cheerfully serving the cached "no such name" for the rest of its fifteen-minute window.

That last point is the whole story. Once a single early query had poisoned the cache, **every subsequent attempt returned the same stale "doesn't exist" answer for up to fifteen minutes** — no matter what I changed in the code. The build wasn't broken; it was being lied to by its own resolver. And because the cache eventually expired, the thing would mysteriously start working again later, which is precisely the behaviour that makes you doubt your own sanity.

## Confirming it with dig

The proof was a single command. Querying a name I knew didn't exist and reading the authority section:

```
;; AUTHORITY SECTION:
example.com.  422  IN  SOA  ns-xxxx.awsdns-xx.co.uk. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400

;; SERVER: 127.0.0.53#53
```

Two things jumped out. The SOA carried the default Route53 values, so the negative-cache window really was 900 seconds — the `422` was just that window counting down. And the server answering was `127.0.0.53`: systemd-resolved, the local stub resolver on the build host itself. The poisoned cache wasn't somewhere upstream and out of reach. It was sitting right on the machine running the build.

## The fix

Two complementary changes.

First, the build process now stops trusting the local resolver for newly-created records. Before any step depends on a fresh hostname, it confirms the record is genuinely live by querying Route53's authoritative name servers directly — authoritative servers never hand back a stale negative — and then clears any negative answer the local resolver had already cached. That removes the race entirely: nothing downstream resolves the name until it actually exists.

Second, I lowered the SOA record's TTL on the zone from 900 seconds to 60. This is defence in depth. Even if some unguarded query slips through in future, a stale negative can now linger for only a minute rather than a quarter of an hour — turning a build-breaking failure into a brief, self-correcting hiccup.

## The lesson

A few things I'll carry away from this:

- **Propagation and caching are different problems.** Authoritative propagation is fast; the delay you actually feel is usually something downstream caching an answer. Don't let the "24 hours" folklore send you chasing the wrong thing.
- **Negative caching is the quiet one.** We all instinctively reason about TTLs on records that exist. The TTL on the *absence* of a record is just as real, it's governed by the SOA, and it's the one that bites automated pipelines that create-then-resolve.
- **When retrying makes it worse, suspect a cache.** A bug that heals itself after a fixed interval, regardless of what you change, is practically a signature for a cache holding a stale answer.
- **Check what your resolver is actually doing.** One `dig`, and specifically the `SERVER:` line and the authority section, told me more than an afternoon of editing build code.

The most humbling bugs are the ones where the code was innocent all along. This was one of them — and now the pipeline waits for the truth before it carries on.