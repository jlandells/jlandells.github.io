---
layout: post
title: "I Replaced Pipedream with Self-Hosted n8n: Here's What I Learned"
date: 2026-06-19 07:34:00 +0100
categories: automation n8n notion todoist
---

I'm not one of those people who tries to bend a single tool to every purpose. I'd rather use the **right** tool for the job and let each one do what it's genuinely good at. For a while, the glue holding my "right tools" together was Pipedream. This is the story of why I tore that out and rebuilt the whole thing on a self-hosted n8n instance running quietly on a VM at home — and the handful of things that bit me along the way.

It's really two stories wearing one coat: a platform switch, and a data-model rebuild that happened to land at the same time. They reinforced each other, so I'll tell both.

## The setup

Each of my customers has a page in Notion. That page shows, at a glance, the outstanding tasks for that customer. Those tasks aren't typed in directly — they're born on a **Meeting Notes** page for a specific meeting, which keeps every open action tied to the conversation that created it. That context matters to me, and Notion is excellent at storing and presenting it.

What Notion isn't is a task manager. It was never designed to be one, and trying to make it behave like one is a fool's errand. Todoist *is* a task manager, and a very good one. So the division of labour is obvious: Notion owns the information, Todoist owns the doing.

The integration's job is to keep those two honest with each other:

- A new task created in a meeting starts life as **Not started** in Notion.
- An automation spots it, creates the matching task in Todoist, and flips the Notion status to **Added to Todoist**.
- The Todoist task carries the URL of the originating Notion task in its description — handy for jumping back, and, more importantly, the thread that lets the return journey work.
- When I tick the task off in Todoist, a second automation notices and sets the Notion task to **Done**.

That's the contract. The interesting part is everything that went wrong with the first attempt at fulfilling it.

## Pipedream: death by a thousand papercuts

The initial build in Pipedream wasn't bad at all. At that point I was using **Sections** under a single parent project for each customer, and Pipedream did a perfectly respectable job of shuttling tasks back and forth.

Then the papercuts started. Every so often I'd get an email telling me a workflow had failed, and working out *why* was often a genuine struggle — the failure was rarely as informative as I'd have liked.

The first real escalation was external. One day **every** workflow started failing at once. The cause turned out to be Todoist overhauling their API, which left my workflows talking to endpoints that no longer behaved as expected. I had to upgrade both. One was civilised about it and offered an "Upgrade" button. The other made me re-create blocks by hand using the new versions. Tedious, but survivable.

The second escalation was the one that properly soured me. Shortly afterwards, the failure emails came back — this time because the workflows were running out of memory. The maddening part was that the *same* workflow had run the day before without needing any extra memory at all. So I dutifully bumped the allocation up. A few days later I had to bump it again. Nothing about what the workflow did had changed; it was simply, and inexplicably, demanding more memory over time — and each increase nudged me closer to the point where I'd have to start paying for a subscription to keep my own task list in sync.

## The final straw

The thing that actually tipped me over wasn't even the flakiness. I'd picked up a new customer and went to add a section for them — only to discover that Todoist caps the number of sections at 20. I'd hit the ceiling.

In fairness, by that point the sections approach was already creaking. With that many of them under one project, I'd lost the at-a-glance overview I'd built the whole thing for. So the cap was less a disaster and more a nudge towards a decision I'd been circling anyway: give **each customer their own project**, all sitting beneath a single parent project, and get my overview back.

A model change of that size meant a rebuild regardless. Given how little faith I had left in Pipedream, I decided not to rebuild *there*. I'd start afresh in my local, self-hosted n8n instance instead.

## The new architecture

The rebuild came out as three workflows and two small data tables, split cleanly by responsibility: one inbound, one outbound, one for maintenance.

### Inbound — New Task from Notion

This workflow polls Notion every minute. When it finds a task sitting at **Not started**, it pulls the relevant customer and task records, looks up the customer's Todoist project, and creates the task there — dropping the Notion task's URL into the Todoist description so the return trip has something to grab. It then sets the Notion status to **Added to Todoist**.

The bit I'm quietly pleased with is the self-healing branch: if the customer's project doesn't yet exist in Todoist, the workflow creates it on the fly, records it in the internal data table, and then carries on to create the task. New customer, no manual setup — it just works.

![The New Task from Notion workflow in n8n](/assets/images/n8n-new-task-from-notion.png)

### Outbound — Update Notion from Todoist

The return journey polls Todoist every minute, looking for changes. Rather than dragging back the entire task list each time, it uses Todoist's sync endpoint together with a stored sync token, so each poll returns only what's actually changed since the last one. The token lives in a data table and gets advanced on every run.

When it finds a change on one of my customer tasks, it checks whether the task has been completed — and if so, sets the corresponding task in Notion to **Done**.

![The Update Notion from Todoist workflow in n8n](/assets/images/n8n-update-notion-from-todoist.png)

### Maintenance — Todoist Project ID Refresh

To translate between customer names and Todoist project IDs, I keep a cache in a data table and refresh it hourly. The refresh walks every project, picks out the ones that are children of the main parent project, and stores each customer's name and project ID alongside a `last_refresh` timestamp. It also prunes any customers that have since been deleted in Todoist, so the cache doesn't drift away from reality.  This approach means we're not having to make dozens of API calls in the other workflows.

![The Todoist Project ID Refresh workflow in n8n](/assets/images/n8n-project-id-refresh.png)

### The two data tables

Behind all three sits state that has to survive between runs — the thing Pipedream always made awkward:

- **`sync_state`** holds the Todoist sync token, so the outbound workflow can do incremental syncs instead of brute-forcing the whole list.
- **`mattermost_projects`** is the name-to-ID cache the refresh keeps current, and the table the inbound workflow writes to when it creates a project for a brand-new customer.

## The quirks of building on a self-hosted box

Moving to my own instance fixed the things that drove me out of Pipedream, but self-hosting brought its own wrinkles. None were fatal; all were instructive.

**No public IP, no OAuth.** My n8n box sits on a home network with no public IP address, which means there's nowhere for an OAuth callback to land. That ruled out the tidy OAuth flow for Todoist. The fix was straightforward enough — authenticate with a personal API token in a Bearer header.

**The bundled Todoist node had fallen behind.** When I started, the built-in Todoist node didn't seem to be keeping up with the current API. I'll be honest, though: I've since discovered I was a few minor versions behind on n8n itself, which may well have been the real culprit — I haven't gone back to retest the node since upgrading to the latest release. Either way, the workaround was painless: Todoist's REST API is well documented, so I simply drove the whole Todoist side through direct HTTP requests. It works, it's transparent, and I can see exactly what's going over the wire.

**One minute, for free.** This one's pure self-hosting payoff. Pipedream wouldn't let me poll more frequently than every 15 minutes without paying. On my own instance, I poll Notion and Todoist every single minute at no cost, so a task I jot down in a meeting is in Todoist before I've closed the laptop. No surprise memory bills, either.

**A harmless date-format wrinkle.** A small one worth noting for anyone doing similar. Generating an ISO timestamp in a Code node produces a slightly different string format from the one you get when an "Insert row" block writes the timestamp for you. It looks untidy side by side, but it doesn't actually matter here — the next refresh overwrites `last_refresh` with a consistent format anyway, so the two converge on their own.

## What I'd tell my past self

A few things crystallised out of all this:

- **Use the right tool, and let the glue be glue.** Notion for information, Todoist for tasks, automation to keep them in step. The moment I stopped asking one tool to be everything, each problem got simpler.
- **Own your runtime.** Self-hosting cost me OAuth convenience and handed me a couple of quirks to work around — but in exchange I got minute-level polling for nothing and, crucially, no inexplicable memory creep slowly herding me towards a paywall.
- **A well-documented API beats a stale node.** When the bundled integration wasn't cooperating, dropping to direct HTTP was quicker than waiting and gave me full visibility into every call.
- **Persist your state deliberately.** Two small data tables — a sync token and a name-to-ID cache — are what let the whole thing run incrementally and survive restarts.

The new setup has been running happily ever since, and the overview I lost to twenty-odd sections is well and truly back. Sometimes the best outcome of a tool letting you down is the rebuild it forces you into.