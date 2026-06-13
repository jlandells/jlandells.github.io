---
layout: post
title: "Testing What You Can't Test First"
date: 2026-06-13 16:28:00 +0100
categories: demo testing
---

The conventional wisdom is that you write your tests first. Red, green, refactor. It's a lovely discipline, and when the feedback loop is measured in seconds it genuinely changes how you build. Test-driven development assumes you can write a failing test, make it pass, and do the whole dance dozens of times an hour.

Our SA demo doesn't work like that.

Before there's anything to test, we've built AMIs and run a `terraform deploy`. The feedback loop isn't seconds; it's coffee breaks. You can't red-green-refactor your way through an infrastructure stand-up, because the "compile" step is spinning up real machines. TDD quietly assumes a cheap, fast loop, and ours is neither.

So rather than forcing a doctrine that doesn't fit, we lean on three layers of testing instead. Each one catches a different class of problem, and none of them on its own is enough.

One thing worth saying up front: this isn't about testing Mattermost. Mattermost has its own testing, and that's not my job. This is about the environment we build *around* it — the scaffolding that makes the demos work. When I say "the demo", I mean our stuff, not the product.

## The sniff test

This is the first line of defence, and it's become far more important since AI joined the development process.

The trouble with AI-generated code and config is that it's *fluent*. It reads well, it usually runs, and it carries an air of confidence that's entirely unearned. Fluency is not correctness. The sniff test is the human pass that asks one simple question: does this smell right? Does anything catch my eye as fishy and worth a closer look?

It's the variable named slightly oddly, the resource quietly placed in the wrong region, the hardcoded value that should have been a variable, the diff that's three times larger than the change you actually asked for. None of it is rigorous. It costs seconds, not minutes, and it catches the obvious-in-hindsight problems before you've spent the better part of an hour deploying something daft.

The honest bit: it's the easiest step to skip when you're moving quickly, and every single time I skip it, I regret it. A few seconds of "hang on, that doesn't look right" saves a deploy cycle you'll never get back.

## Automated tests

This is the backbone. We've got a full suite that tries to cover the most important use cases across our demos, and it takes about an hour to run.

That hour isn't a failing — it's the cost of the loop. When your tests are exercising something close to a real environment rather than a handful of mocked functions, an hour is what honesty costs. The trade is that you get to trust a change you couldn't possibly eyeball your way through.

The interesting thing is *how* the suite earns its keep. It's found genuinely subtle bugs in two quite different ways. The first is the obvious one: a test fails and points straight at a real problem. The second is the underrated one: a test fails when it *shouldn't* have. A false negative like that isn't noise — it's telling you that the test's assumptions don't match reality, which means the test is too brittle, or simply wrong. Chasing those down has let us tighten the suite considerably, and a test that fails for the right reason next time is worth more than one that's never been stress-tested.

A test you trust is one that's been wrong at least once and been fixed.

## Smoke tests

The final step, and the one nothing else can replace: a manual run-through covering a solid cross-section of what we actually use in the daily demos.

You'd think a full automated suite would make this redundant. It doesn't, and the reason is simple — automated tests only check what you told them to check. They're brilliant at catching the things you thought to assert and useless at everything else. The manual pass catches the rest: the flow that feels off, the screen that's mysteriously slow, the thing that's visually wrong, the bit a customer will actually notice the moment you put it in front of them.

It's really the "would I be happy demoing this tomorrow?" test. And the only way to answer that question honestly is to sit down and drive the thing.

## Why three, and in this order

These aren't a hierarchy you climb, with the "best" one at the top. They're layers, and they catch different things at different costs.

The sniff test is cheap and fast, and it stops fluent nonsense before it ever reaches a deploy. The automated suite is expensive — an hour of it — but it catches the regressions and subtle behaviour no human is going to spot by reading a diff. The smoke test catches the lived-experience problems that no assertion ever thought to cover.

TDD is a fine discipline when the loop is fast and cheap. When your loop is an AMI build and a `terraform deploy`, you adapt to what you've actually got. This is that adaptation — and future me, when you're staring at this in three years wondering why we bother with all three: it's because each one has, at some point, caught something the other two missed.