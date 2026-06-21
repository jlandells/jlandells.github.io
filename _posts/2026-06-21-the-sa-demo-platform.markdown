---
layout: post
title: "The SA Demo Platform: How It Grew, and What It Taught Me"
date: 2026-06-21 02:30:00 +0100
categories: mattermost demo platform architecture
---

When I joined Mattermost, one of my onboarding tasks was to stand up a copy of the SA demo environment and have a poke around, so I'd get comfortable with the Terraform we used to build it. A perfectly sensible exercise. What nobody mentioned — least of all me, because I hadn't the faintest idea — was that I'd end up owning the whole thing.

This is the story of how a blank, three-server demo became the platform we now use to tell Mattermost's most demanding stories: live integrations with security and defence partners, simulated operational scenarios, AI in the loop, and the ability for anyone in the company to spin up their own copy in minutes. It's also a story about decisions — the ones that worked, the ones that cost me, and the ones I'd defend to anybody who asked.

Consider this the overview: the *why*, told from altitude. Where there's a deeper account worth having, I've linked it. The rest will follow as I write it.

## What I inherited

The environment had been built a few years before I arrived as a way to spin up an independent demo on demand. Terraform, a handful of shell scripts and some Python, deploying into AWS, and you'd get three servers: Mattermost, PostgreSQL and OpenLDAP. That was it — a blank instance you could use to show the features off.

A few people used it regularly. They'd spin one up, then leave it running indefinitely, upgrading bits and pieces by hand when they remembered to. And the version of Mattermost it deployed was around two years out of date.

So the first two things I did set the tone for everything that followed. I brought the deployed version current and kept it tracking our monthly releases — because a demo running two-year-old software isn't a demo, it's a liability. And I started adding content. Having spent most of my career in pre-sales, I knew a blank instance sells nothing; you need a *value-based* demo, something with a story in it. That instinct — that the platform exists to tell a story, not to list features — turned out to be the thread running through every decision since.

## The incident that changed how I work

Early on, a colleague added a metrics server and an RTCD server (for Calls) to the build. Useful additions — except I discovered them when my next pull request hit dozens of merge conflicts. He'd used AI with no constraints whatsoever, and it had touched an enormous number of lines to achieve a fairly modest result. Cleaning that up took me a good while.

Around the same time, a security audit flagged that we had SSH wide open to the internet, and I designed a scheme to give each demo its own DNS name rather than relying on IP addresses that changed on every build.

The upshot was that I took on sole ownership, and we locked PR approval down to just two of us. That decision had a cost — two approvers is a thin bus factor, and it can slow a merge when we're both busy — but the platform had reached the point where coherence mattered more than throughput, and uncontrolled change was the bigger risk. I'd make the same call again.

Hold that AI story in mind, though. It comes back.

## Getting a direction

About then, the company's focus sharpened from being a general-purpose collaboration tool to being a collaboration hub for Defence, Intelligence, Security and Critical Infrastructure. Suddenly I had the direction I needed to build a *proper* demo.

The approach was to create multiple teams within Mattermost, each focused on an industry, each with realistic channels, realistic conversations, and — for the flagship defence team, "Joint Operations Command" — right down to a realistic military colour theme. To make the scenarios live, we built a Demo Control Panel (a Flask server running on the Mattermost host) that could trigger inbound events looking for all the world like they'd come from real third-party integrations.

The piece I'm proudest of from this era is the Playbooks work. Rather than simply exporting a few Playbooks, I wrote a specification that *simulated* completed and partially completed runs. That mattered enormously, because it let us tell the end-to-end story — and, crucially, show Playbook metrics varying over time. You cannot demonstrate a time-series feature in an instance that was built ten minutes ago unless you manufacture its history, and that's exactly what we did.

## The AI story, revisited

I'd been asked to include Boards, but the public API was sorely lacking, and every attempt I made to drive it programmatically did nothing but crash my builds entirely.

This is where Claude Code came in — and yes, I was cautious, given what unconstrained AI had done to my merge queue not long before. But used *with* constraints, pointed at the source on GitHub purely to build my own understanding of the APIs, it was transformative. There was still plenty of to and fro — cross-referencing the Chrome developer tools against specific calls in Postman to confirm our theories — but we got there. In the end we could hand the system a single JSON file describing multiple Boards across multiple demos, and have it build them all at deploy time.

Same tool, opposite outcome. The difference was entirely the constraints. That's not a coincidence; it's the whole lesson.

## The build-time war

By now we were closing on ten teams (we're at fourteen today), and the build was taking well over forty minutes. That's a long time to wait to find out whether you've broken something.

The realisation that unlocked it was simple: the PostgreSQL and OpenLDAP servers never changed. We were rebuilding identical machines on every single deploy. So I used Packer to bake those two into AMIs, which brought us to around thirty minutes. Still too long. So I redesigned the Mattermost deployment the same way — pre-loading *everything* into the AMI: the server, the plugins, the images, the JSON files, the scripts. No more downloading the world on every build. That took us to roughly fifteen minutes.

The trade-off is real and I chose it deliberately: the platform is now fast to deploy and deliberately slow to *change*, because changing anything means rebuilding an image. The person who pays that price, by design, is me — and I've [written separately about that bargain and what happens when an AI doesn't see the whole picture]({{ site.url }}/terraform/packer/ai/development/2026-06-12-slow-feedback-loops.html). It also reshaped how I think about testing something whose "compile step" is spinning up real machines, which is [a discipline of its own]({{ site.url }}/demo/testing/2026-06-13-testing-our-demo-builds.html). And occasionally the slowness isn't the design at all but [something far more insidious hiding in the infrastructure]({{ site.url }}/infrastructure/networking/dns/2026-06-10-dns-negative-caching.html).

## Letting people help themselves

A goal I'd held from early on was to let *anyone* — not just the technically confident — spin up their own demo. So I built a Rundeck server with four user classes mapped to four projects: Admin, Technical Users, Non-Technical Users, and Partners.

Rundeck's built-in authentication is weak (I'm being kind), so I deployed Keycloak to give people a more production-like way to manage their own credentials. That was several days of genuine frustration before it behaved, but it was worth it. We also built in automatic termination of every demo after a set number of hours, to keep AWS spend under control — there's no sense in a demo sitting idle all night and all weekend because someone needed it for two hours on a Tuesday afternoon.

Auto-termination is another deliberate trade-off with a sharp edge: get it wrong and someone loses a demo they were halfway through preparing. But an unbounded AWS bill was the larger danger, and a predictable teardown is easier to plan around than a nasty invoice.

## The Dataminr Bridge

When I was asked to build a *real* integration with Dataminr First Alert, the API itself turned out to be refreshingly straightforward. The one caution the Dataminr team gave me was about fairly restrictive rate limiting — and with potentially dozens of demo instances running at once, I could see most of them tripping over those limits and ruining the experience.

So we built what we call the Dataminr Bridge: an EC2 instance that polls Dataminr once every thirty seconds, caches new events in a SQLite database, and exposes its own API back into our environment. Each demo registers its inbound webhook URL with the Bridge, and whenever new events land, we fan them out to everyone who's registered. One polite consumer of the official API; as many demos behind it as we like.

There's a fair amount of cleanup machinery in there too, and the whole pattern — putting a cache between yourself and a rate-limited third party — deserves a post of its own. It's on the list.

## Pexip, and then Demo 2.0

Before any of the most recent work, we stood up a dedicated Pexip environment for showing that integration. It added another node to the build — and it's the one node we *can't* bake into our own AMI, since Pexip supply theirs — but it earned its place for the sake of the story.

What came next is what we're calling Demo 2.0, our most recent major chapter. The point of it is to make the defence story genuinely compelling rather than merely plausible:

- **archTIS** for enhancing permissions with attribute-based controls;
- **Arqit** to add post-quantum encryption to specific channels, restricting which devices can see what;
- **Whitespace**, whose AI produces powerful after-action reports from an actual Playbook run;
- and **Duality** (simulated, building on their own proof-of-concept) for Zero Footprint Interrogation — letting a high-side deployment query low-side data sources without leaving a trace.

Stitching those scenarios together meant adding an n8n server to every build, deployed from its own custom AMI, which is [a story I've already told]({{ site.url }}/mattermost/automation/n8n/ai/2026-06-15-why-n8n-powers-our-demo.html). Keeping the AI agents themselves configured correctly across freshly-built environments turned into [its own field guide to an undocumented API]({{ site.url }}/mattermost/ai/agents/plugin/api/2026-06-11-agents-plugin-api.html) after a breaking change quietly stopped our old approach working.

All of that pushed the build time back up to around twenty-three minutes — and yes, I've got ideas for clawing some of that back. But for everything it now does, it's working really well.

The final piece is DemoDocs, a BookStack server that uses the same credentials people already have for Rundeck, so we can restrict individual books to specific groups. It holds the living documentation and the training walkthroughs — because a platform nobody understands isn't really a platform at all.

## The pattern, if there is one

Looking back over all of it, there's a shape that repeats. Every meaningful leap forward was forced by a constraint — stale software, no content, unconstrained AI, forty-minute builds, an aggressive rate limit, weak authentication, runaway costs — and met with a considered response that cost something. The art was never in avoiding the trade-offs. It was in choosing which price was worth paying, and being honest about it.

That's the platform. The posts below go deeper into the individual decisions, and I'll add to them as I write.

## Deep dives

- [Fast to Deploy, Slow to Change]({{ site.url }}/terraform/packer/ai/development/2026-06-12-slow-feedback-loops.html) — the AMI-everything bargain, and the price I pay for it.
- [Testing What You Can't Test First]({{ site.url }}/demo/testing/2026-06-13-testing-our-demo-builds.html) — how you test something whose compile step is real machines.
- [It Wasn't the Code: DNS Negative Caching]({{ site.url }}/infrastructure/networking/dns/2026-06-10-dns-negative-caching.html) — when the build breaks and the build logs lie to you.
- [Why n8n Now Powers Our Demo Platform's Slash Commands]({{ site.url }}/mattermost/automation/n8n/ai/2026-06-15-why-n8n-powers-our-demo.html) — moving the orchestration off ad-hoc Python.
- [Provisioning Mattermost AI Agents by API]({{ site.url }}/mattermost/ai/agents/plugin/api/2026-06-11-agents-plugin-api.html) — a field guide to the Agents plugin's undocumented REST interface.

*Still to come: the Dataminr Bridge, the Playbooks simulation specification, and a few of the technical wins I glossed over here.*
