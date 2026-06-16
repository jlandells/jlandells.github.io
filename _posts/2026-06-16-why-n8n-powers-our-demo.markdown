---
layout: post
title: "Why n8n Now Powers Our Demo Platform's Slash Commands"
date: 2026-06-16 16:53:00 +0100
categories: mattermost automation n8n ai
---

As part of my role at Mattermost, I look after our sales demo platform — the environment we use to show prospects what Mattermost can do, and a substantial piece of engineering in its own right. For a good while, the slash commands behind it were Python scripts. They did the job. A command came in, a script ran, something useful came back. For anything self-contained — provisioning a bit of content, nudging an integration, posting a canned response — that arrangement was perfectly serviceable, and for the simple cases it still is.

The trouble started the moment we wanted to put AI in the loop. A demo platform whose whole purpose is to show Mattermost off as AI-augmented collaboration infrastructure reaches that moment quickly. Wiring an HTTP call to a model endpoint into a Python script is easy enough to write. Doing it *well* — with retries when the model returns something malformed, validation before you act on the response, and somewhere sensible to inspect what came back when it inevitably misbehaves during a live demo — is a different proposition entirely. Each script grew a little ecosystem of plumbing around the actual interesting part, and that plumbing was different every time.

n8n changed how I think about the whole problem.

## It's about the shape of the data

For anyone who hasn't come across it: n8n is a workflow automation tool, self-hostable, where you build flows out of nodes. A trigger fires, data moves from node to node, things happen. On paper that's nothing new.

What won me over wasn't a feature. It was the shape of how data moves through a workflow. A request lands on a Webhook node carrying its JSON payload, and from that point every downstream node can see the entire accumulated structure — not merely the output of the node immediately upstream. You enrich the payload as you go, and at the far end you can still reach all the way back and pull out something you gathered near the beginning. Build on it, reference any part of it, reshape it into a response. Once that clicks, a great many fiddly integration problems collapse into something you can lay out visually and reason about.

## A worked example: the Incident Summary command

The workflow I'm fondest of runs at the end of a Playbook Run.

![The Incident Summary workflow in n8n](/assets/images/n8n-incident-summary.png)
*One webhook in; a posted summary, threaded detail, and an actionable card out.*

The premise is simple to describe and was previously rather painful to implement. At the close of an incident, you fire the slash command in the run channel. The workflow walks back through that channel, gathers every post and every file attachment, and bundles the lot into a single structure. That bundle becomes the prompt context sent to Anthropic, which produces a structured Incident Summary. The summary is posted back into the channel, broken into its constituent sections — each posted as a threaded reply so the detail stays tidy — and rounded off with a card carrying a button. Click it, and the summary is handed to a third-party system that generates a formal After Action Report. That handoff, naturally, is another n8n workflow.

Tracing the flow makes the data-shape point concrete:

- The **Webhook** receives the command and its context — who ran it, in which channel, on which team.
- **Build Bundle** walks the channel and packages every post and attachment into a single structure.
- An **If Success** guard decides whether there's anything worth summarising. If not, it responds with a polite refusal; if so, it fires an immediate *in progress* acknowledgement — slash commands time out, and the model call is not quick.
- **Build Claude Request** assembles the payload, **Anthropic Generate** makes the call, and **Parse Summary** turns the response into something structured.
- A **Summary Valid?** check stands between the model and anything user-facing. If the response doesn't hold up, a **Retries Left?** node either loops back for another attempt or, having exhausted them, posts a failure notice rather than something half-formed.
- Only once the summary is sound does it get posted, split into sections, looped over to post each as a thread, and finished with the action card.

Build Bundle is worth dwelling on for a moment: it isn't native n8n conjuring, but a Python script running on the n8n host. n8n hasn't made Python redundant — it has given it a tidier role, orchestrating the heavy data-gathering rather than asking the workflow canvas to carry the whole job alone.

The part that still pleases me: the original webhook context — channel, team, user — is just *there* at the posting stage, a dozen nodes downstream, ready to be referenced. I never had to thread it carefully through a chain of function calls. It travelled with the workflow.

## The AI seam that isn't one

Third on my own list of why this works: putting AI into a workflow is, more or less, dropping a single node in and pointing it at Anthropic's endpoint. The model call sits among the other nodes as an equal — not a special case bolted on with its own bespoke error handling, but one step in a flow that already knows how to validate, retry, and branch. The thing that used to be the awkward part of a Python script becomes the least remarkable node on the canvas. That inversion is the whole point.

## Debugging without the wait

The feature I'd happily evangelise on its own is the ability to pin a node's output. My habit is to pin the initial Webhook: freeze one representative payload, and the whole workflow becomes replayable at will. No ending a Playbook Run and firing a real slash command from Mattermost just to get test data flowing — I can walk through every step that follows, as many times as I like, from a known starting point.

From there you can pin further down as the need arises. Freeze the Anthropic response while you fiddle with how the summary is parsed, split, and posted, and you iterate on the formatting without burning a fresh model call each time. Better still, you can see at a glance exactly what each node received and emitted, which turns "why did that come out wrong" from an archaeology exercise into something you can read off the screen. It has saved me an enormous amount of time and more than a little frustration.

## The honest bits

It isn't all effortless. Because everything we deploy goes up through Terraform, the workflows themselves have to be built as code at spin-up rather than clicked together by hand. Getting that reliable took a great deal of trial and error — exporting, templating, and re-importing workflows so they come up identically every time is not where n8n is at its most forgiving. It runs smoothly now, but I won't pretend the road there was short.

I'm also not religious about it. We already have a Flask server running, and for the simple, AI-free commands it handles them cleanly with far less ceremony. n8n earns its place on the complex workflows — the ones with branching, validation, retries, and a model in the middle. Reaching for it to echo a string back would be using a rather nice hammer on a drawing pin.

## Where it leaves us

The slash command used to be a thin wrapper around a script. With n8n it has become a place where real work happens — where a request can gather context, consult a model, check the model's homework, and respond with something worth having, all in a flow you can see and reason about end to end. For a platform whose entire job is to demonstrate that collaboration tooling can be more than a chat box, that turns out to matter rather a lot.