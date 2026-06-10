---
layout: post
title:  "Slow Feedback Loops"
date:   2026-06-12 09:00:00 +0100
categories: terraform packer ai development
---

# Fast to Deploy, Slow to Change

*The trade-off I chose, the price I pay for it, and what happens when an AI doesn't see the whole picture*

Every system is optimised for something, whether you decided it deliberately or not. The platform I spend most of my days on is optimised, hard, for one thing: spinning up fast and reliably, on demand. It does that well. The price is that it is deliberately, painfully slow to *change* — and the person who pays that price, by design, is me.

This is a post about a trade-off I'd make again tomorrow, and about what it's actually like to live inside it once you add an AI assistant to the mix.

## The right call

The platform stands up a complete, working environment from nothing. Early on, it did that the obvious way: provision everything at deploy time with Terraform. The trouble was the heavy pieces. A fully configured PostgreSQL instance and an OpenLDAP server don't appear instantly — and standing them up freshly on *every single deployment* made the whole thing crawl.

So I stopped doing that work on every deploy and did it once instead, at image-build time. Bake PostgreSQL and OpenLDAP — configured, ready — straight into the AMI with Packer, and let each deployment start from an image that's already most of the way there. Deployment times dropped dramatically.

For a platform whose entire reason to exist is to come up convincingly and reliably when someone needs it, that isn't a nice-to-have. It's the point. Fast deployment wins every argument, because fast deployment is the product.

## Someone pays, and it's me

Here's the bit that doesn't show up in the demo. Baking work into the image speeds up *deployment* — but it taxes *iteration*, and iteration is what I actually do all day.

Because the thing I spend my time on is tweaking the build itself, almost every change I make lives inside that baked image. And a change inside the image means there's no quick way to see if it worked. I have to run Packer to rebuild the AMI, then deploy the whole stack on top of it, before I can lay eyes on the result. That's about forty-five minutes. Every time.

I chose that. The cost of a fast, reliable deploy didn't vanish — it moved. It moved off the demo experience and onto the maintainer, which is exactly where it should sit. The people the platform is *for* get the speed; I absorb the slowness on their behalf. That's the deal, and it's the correct deal. It's just that "correct" and "comfortable" are rarely the same word.

## The penalty loop

Forty-five minutes is the routine tax. The real reckoning comes later.

I only run the full test suite when a change feels ready to become a pull request — and that takes about an hour. So the moment of truth is deliberately deferred to the end. Which means that when the suite finds a bug, I'm not fixing a line and re-running a quick check. I'm going back to the very beginning: bake a new AMI, deploy the stack, run the hour again. One fault discovered late doesn't cost me a fix. It costs me a rebuild *and* a re-test, and if the fix isn't right first time, it costs me both again.

This is the loop that teaches you humility. On a fast feedback loop, being wrong is cheap and frequent and nobody minds. On this one, being wrong has a price tag you can feel.

## Enter the AI

I delegate a lot of the implementation to Claude Code, and when it's right it's a genuine multiplier — it does the work while I think about the next thing. But it changes *where* mistakes come from, and on a slow loop that matters enormously.

The platform is complex. Not complicated-looking; genuinely complex — a dense web of interdependent components where pulling one thread tightens three others somewhere you weren't looking. I carry most of that interdependency map around in my head, because I built it. Claude Code doesn't. And its characteristic failure isn't getting the local task wrong — it's usually very good at the local task. It's making a change that's perfectly sensible *in isolation*, presenting it with complete confidence, and not noticing that it's just quietly broken something three steps away that it never had in view.

In a simple codebase, that's harmless; you spot it instantly and move on. Here, "confident and locally correct" is precisely the combination that hurts. The confidence discourages me from double-checking. The complexity means the damage lands somewhere non-obvious. And the loop is too slow to catch it cheaply — so I find out about the collateral breakage forty-five minutes later, or worse, an hour later at test time. The change didn't cost me a line. It cost me the full penalty.

## What actually helps

You can't make the loop fast, so you learn to spend it carefully.

- **Treat the cycle as the expensive currency.** Minutes spent reading, questioning and sanity-checking a change are minutes; a wasted rebuild is the better part of an hour. Front-load the scrutiny — it's nearly free by comparison.
- **Lend the assistant the context it lacks.** It doesn't hold the interdependency map, so I hand it the relevant parts up front: what this touches, what mustn't break, what's coupled to what. The institutional knowledge is mine to supply, not its to guess.
- **Make it think about blast radius before it reaches for a fix.** "What else could this change affect?" asked *before* a rebuild is worth far more than the same question answered by a failed test.
- **Manufacture cheaper checks.** Anything that can fail in five minutes instead of forty-five is worth building — a quick local validation, a partial check — so the full Packer run only happens when I genuinely expect it to pass.
- **Keep the judgement for myself.** The assistant implements; deciding whether a proposed change is even *plausible* in light of the whole system, before I spend the cycle proving it isn't, is the part that's still mine. It's also the part that pays.

## The price was always going to be paid by someone

Good infrastructure decisions rarely make the pain disappear. They move it — and the skill is moving it onto whoever's best placed to absorb it. For this platform, that's the maintainer, deliberately, so that the thing everyone else relies on stays fast and dependable. I knew the bill when I signed up for it.

What I've learned is that the discipline isn't in resenting that bill; it's in not letting anything inflate it past the figure I already accepted. A confident answer made without the full picture — whether it's mine or an AI's — is the most expensive thing on a slow loop, right up until the moment it's actually been tested. So I test sooner where I can, I think harder before I commit a cycle, and I never mistake confidence for correctness.

The deployment is fast. I suffer for it. That part was always the plan.