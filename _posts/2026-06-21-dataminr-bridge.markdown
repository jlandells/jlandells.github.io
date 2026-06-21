---
layout: post
title: "The Dataminr Bridge: Feeding One Rate-Limited API to a Fleet of Demos"
date: 2026-06-21 17:45:00 +0100
categories: mattermost demo dataminr aws integration
---

When I was asked to build a *real* integration with Dataminr First Alert, the API itself turned out to be a pleasure. The Dataminr team set me up on the spot and let me loose, with a single word of caution: the alerts endpoint is rate-limited, and the limit applies *per account* — and I had exactly one account to work with.

On its own, that wouldn't matter. One demo polling for breaking-news alerts on a sensible cadence is no trouble at all. The problem is that I don't have one demo. On a busy day I have a dozen or more, each a freshly-built, self-service environment that someone span up for a couple of hours — and every one of them would have been drawing alerts down from that *same single budget*. It wouldn't take many before they were tripping over one another into rate-limit errors, and a demo whose "live" integration has been throttled into silence is worse than no integration at all.

So the question wasn't "how do I call Dataminr?" It was "how do I let *many* ephemeral demos appear to have a live Dataminr feed, while only ever being one well-behaved consumer of the actual API?" The answer is a small service we call the Dataminr Bridge, and the pattern behind it is far more general than Dataminr.

This is one of the deep-dives behind [the SA Demo Platform overview]({{ site.url }}/mattermost/demo/platform/architecture/2026-06-21-the-sa-demo-platform.html); you don't need to have read that to follow this, but it sets the wider context.

## One polite consumer, everyone else behind it

The shape of the solution is the oldest trick there is: put a cache in the middle. A single service polls Dataminr on a steady cadence, stores what it sees, and fans new events out to anyone who's registered an interest. Dataminr only ever talks to one client, on one predictable schedule. The demos talk to the Bridge, which has no rate limit because it's mine.

Concretely, the Bridge is a Flask application on a single EC2 instance, sitting behind an Application Load Balancer with a Route53 name, and backed by a SQLite database on an attached EBS volume. The whole thing is baked into an AMI with Packer and deployed with Terraform, exactly like every other component of the platform — so rebuilding it is a known quantity rather than an adventure.

The choice of SQLite usually raises an eyebrow, so let me defend it before anyone asks. This is a single-node service with a single writer: one poller putting events in, a thread pool reading them out. There's no horizontal scaling story to tell, and there doesn't need to be. SQLite on a durable EBS volume gives me persistence across restarts, transactional safety, and precisely zero additional infrastructure to provision, secure, or pay for. A managed database here would be cost and ceremony in exchange for nothing I need. The cost of the decision is honest enough — the Bridge cannot scale out, and if it falls over, the demos' live feed goes quiet until it's back — but for a demonstration platform that's an acceptable price, and a single tidy EC2 instance is a great deal easier to reason about at three in the morning than a distributed anything.

## Polling without drowning in duplicates

A scheduled job polls Dataminr every thirty seconds. Each event comes back with an identifier, and the events table carries a `UNIQUE(source, alert_id)` constraint, so storing a new batch is simply an `INSERT OR IGNORE` — anything seen before is silently dropped on the floor. No bookkeeping, no "have I seen this?" lookups, just let the database enforce uniqueness and move on.

The poller keeps its place using a small `system_state` table keyed by source, so a restart resumes from where it left off rather than either replaying history or losing the thread. Note the `source` part of all of this: the Bridge was deliberately built around *pluggable* alert sources. Dataminr is the one that's live, but PagerDuty and Splunk sit in the configuration as commented-out examples, each with its own poll interval. The caching-and-fan-out machinery doesn't care where an event came from, which means the next integration is a new source module rather than a new service.

## Fanning out to ephemeral consumers

A demo joins in by calling `POST /api/register` with its Mattermost incoming-webhook URL. When a new event arrives, the Bridge formats it into a Mattermost webhook payload and delivers it to every registered webhook concurrently, through a thread pool — twenty workers, each with a ten-second timeout. A 2xx response counts as success; anything else, or a timeout, counts as a failure. So far, so unremarkable.

The interesting part is what "a registered consumer" actually *means* when your consumers are demos that auto-terminate after a few hours. And that's where the war stories live.

## War story one: the day a demo got the entire back catalogue

Alongside the webhook fan-out, the Bridge also impersonates Dataminr. We needed to develop and test the Mattermost-side plugin without hammering the real API — or needing it at all — so the Bridge exposes a set of `/dm_api/...` endpoints that replicate Dataminr's own authentication and alert-fetch calls, complete with token issuing and cursor-based pagination. A cache that quietly grew into a stunt double.

The first version of that emulation had a flaw that's obvious in hindsight. When a plugin authenticated and asked for alerts *for the first time* — with no cursor yet — the honest answer to "give me everything since the beginning" was, well, everything. The entire cache, dumped in one response. Fine with three test alerts; decidedly not fine once the cache had a few days of real events in it.

The fix was to cap the initial, cursorless fetch to the most recent handful of alerts, after which normal cursor pagination takes over. It's a one-line idea — *the first page is not the whole history* — but it's the kind of thing you only notice when a freshly-connected client suddenly gets a wall of week-old news. First-connect semantics deserve as much thought as steady-state ones.

## War story two: webhooks that outlive their demos

The bigger story is dead webhooks, and it's the one I'd been promising to write up.

Here's the situation. Demos auto-terminate to keep AWS costs sane — a deliberate decision I've described elsewhere — which means a registered webhook URL can stop existing at any moment, without warning, while the Bridge happily keeps trying to deliver to it. Multiply that by a fleet of short-lived demos and you accumulate a graveyard of registrations pointing at servers that are no longer there, every one of them failing on every single event, forever.

So delivery failure had to become a *lifecycle signal* rather than just a logged error. Each registration tracks its consecutive failures. A successful delivery resets that counter to zero. Three consecutive failures, and the registration is marked `dead`, with a `dead_since` timestamp recorded. Dead registrations are skipped, so the fleet's churn stops dragging on every fan-out.

Two refinements make it humane rather than brutal. First, resurrection: if a demo comes *back* — same webhook URL re-registering — the Bridge reactivates the existing dead row instead of creating a duplicate, clearing the failure count and the death certificate in one go. Second, a reaper: a periodic cleanup deletes registrations that have been dead longer than a retention window, so the table doesn't slowly fill with tombstones. Old cached alerts and audit logs get the same treatment on their own schedules.

The lesson generalises well beyond webhooks. The moment your consumers are ephemeral — containers, ephemeral environments, anything that can vanish mid-conversation — you cannot treat a failed delivery as a transient blip to be retried indefinitely. Failure has to *accrue* into a state change: alive, struggling, dead, and possibly alive again. Build that lifecycle in deliberately, or your system will quietly spend more and more of its effort talking to the departed.

## What I'd carry to the next one

Strip away the specifics and the Bridge is one transferable idea wearing several hats:

- **Be a single, polite consumer of anything rate-limited, and fan out from a cache.** The third party sees one well-behaved client on a fixed cadence; your fleet sees an unrestricted local service. The rate limit simply stops being your problem.
- **Let the datastore enforce what it's good at.** A uniqueness constraint and `INSERT OR IGNORE` is a more reliable de-duplicator than any logic I'd write around it.
- **Treat ephemeral consumers as ephemeral.** Delivery failure is a lifecycle event, not an exception. Mark, reactivate, reap.
- **Design the first interaction as carefully as the steady state.** The cursorless first fetch is where the cache-dump bug lived, and it's exactly the case that's easy to skip when everything works with three test records.

The honest costs are worth stating too, because the [platform overview]({{ site.url }}/mattermost/demo/platform/architecture/2026-06-21-the-sa-demo-platform.html) makes the point that every decision has one. The Bridge is a single point of failure for the demos' live feed, it doesn't scale horizontally, and — like rather too much of the platform — it's mine alone to keep running. For a demonstration environment, those are prices I'll happily pay for something this simple to reason about. In production, with real users depending on the feed, I'd be making some of these trades very differently.

---

*This is one of the deep-dives behind [the SA Demo Platform overview]({{ site.url }}/mattermost/demo/platform/architecture/2026-06-21-the-sa-demo-platform.html), where the rest of the series lives.*
