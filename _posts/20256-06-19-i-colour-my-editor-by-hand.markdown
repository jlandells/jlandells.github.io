---
layout: post
title: "I Colour My Editor by Hand, On Purpose"
date: 2026-06-19 16:40:00 +0100
categories: tooling git vscode workflow
---

It's the middle of the afternoon. You've been heads-down for hours, a proper run of focus, the kind that doesn't come along often enough. Half a dozen files touched, the change finally hanging together the way you'd pictured it. You sit back, reach for the keyboard to commit, and your eye drifts to the branch name in the corner.

`main`.

Not a feature branch. Not the branch you actually meant to be on. `main`. The work isn't lost — nothing's broken — but you now have half a day of changes sitting exactly where they shouldn't be, and you've got to shuffle them somewhere proper before you can carry on. If you're entirely comfortable with git, that's a two-minute job. If you're *not* — and plenty of perfectly competent developers quietly aren't — it's a small knot of dread, and the temptation to just commit-where-you-are and sort it out later is real.

It doesn't happen to me often — I'm fairly careful about how I move around in git, and I'll come to that in a moment — but "not often" isn't "never", and the one occasion it does happen is the one occasion it matters. So I built a guardrail. It isn't clever, and that's rather the point.

## The obvious fix (and why I didn't take it)

The instinctive engineering answer is to automate it. Hang a script off a `post-checkout` hook, have it inspect the branch you've just landed on, and recolour the editor accordingly — red for `main`, something calmer for everything else. Set it once, never think about it again. Very tidy.

I considered exactly this. I didn't do it. And the more I sat with the decision, the more I became convinced the manual version is the *better* one — not the compromise, the actual right answer.

## What I actually do

I use [Peacock](https://www.peacockcode.dev/), John Papa's lovely little VSCode extension. Its day job is telling editor windows apart — it subtly recolours the workbench chrome (the title bar, the activity bar, the status bar) so that when you've got four instances open you can tell at a glance which is which. I've bent it to a different purpose entirely: signalling *where I am in git*, not *which window I'm in*.

Three states, three colours:

- **Bright red** — I'm on `main`. This is my home base: it's where I sit *between* pieces of work, and it's the one branch where a careless commit really bites. So it gets the loudest, least comfortable colour, and I stay parked in it until I've made a deliberate decision to start something new.
- **Neutral** — the standard theme. I've cleared the colour and cut a fresh branch off `main`. I'm head-down and free to crack on; the absence of a warning *is* the signal.
- **Orange** — I've raised the pull request. I can still push tweaks, and often do — but the colour nags me to stop treating the branch as my private sandpit. It's under review now, so I keep my reviewer in the loop and ask them to hold off merging until I've finished fiddling, rather than having it merged out from under me.

![The three editor states: bright red on main, orange on a branch with an open PR, and the neutral working theme](/assets/images/vscode-colours.png)

In practice it runs as a loop. I start parked on `main`, in red. When I want to begin something new, I hit `Reset` to clear the colour and cut a branch — then stay neutral, head-down, until the work is ready. At that point I raise the pull request and press `PR` to go orange, my cue to stop pushing changes on autopilot and to keep my reviewer in step. Once I see the PR has been merged, I switch back to `main`, sync it so it's bang up to date, and press the `main` button to turn the editor red once more. And there I sit, back at home base, until the whole thing comes round again — which has the happy side effect that the next branch I cut is always taken from a current `main`, never from some half-finished branch I forgot to leave behind.

The colours themselves are set by a small set of Keyboard Maestro macros, each firing the relevant Peacock command. And because it wouldn't be me otherwise, every one is wired to a Stream Deck key: `main` to go red, `PR` to go orange, and `Reset` to clear back to neutral. The whole dance is a single keypress away, without my hands leaving the desk.

One small wrinkle worth flagging, because it's the sort of thing that catches you out: those macros drive Peacock through VSCode's interface directly, rather than through any tidy programmatic API — and that made them surprisingly sensitive to timing. Fire the steps off too quickly and the UI simply hasn't caught up, so the macro misses its target and nothing happens. I had to sprinkle in a few tiny pauses to let the interface settle between steps. Not elegant, but reliable — and, if I'm honest, a quiet little demonstration of the very thing this post is about: even this much automation has a fiddly edge that needed filing down by hand.

![The three Stream Deck keys, labelled main, PR, and Reset](/assets/images/stream-deck-colours.png)

That's it. No hooks, no scripts, no daemon quietly watching `HEAD`. A human, a button, a colour.

## Why doing it by hand is the whole point

Here's the thing the automated version gets wrong, and it took me a while to articulate it: **the act of pressing the button is the acknowledgement.**

When I deliberately turn the editor red as I check out `main`, I'm not just changing a colour — I'm telling myself, in that exact moment, "you are now somewhere you can do damage; behave accordingly." The cost of the action is the feature, not the bug. A half-second of deliberate effort buys a moment of genuine attention.

An automated recolour does the opposite. It goes red silently, on my behalf, while my mind is elsewhere — and within a fortnight my eye simply stops registering it. It becomes wallpaper. We've all learned not to read the cookie banner; an alert that fires every single time, without my involvement, gets filed in the same mental bin. The signal that's always on is the signal you stop seeing.

Manual keeps the colour *meaning something*, precisely because I had to choose it.

## The bit I won't pretend away

There's an obvious hole in this, and I'd rather name it than have you spot it and think I hadn't: **you can forget to press the button.**

This is not a gate. It does not physically stop me committing to `main`. If I check out `main` and forget to turn it red, the editor stays its calm, reassuring neutral and offers me no warning whatsoever. It's an intention-setting aid, not a fence. Anyone selling you a fence here is selling you the git hook — which, fair enough, will catch the case where you forgot. I've simply decided that the silent, forgettable fence is worth less to me than the loud, deliberate ritual, and I'd rather occasionally miss a press than slowly go blind to a colour.

Your mileage may vary, and if you're the sort who genuinely never remembers, automate it without guilt.

## Getting warier with age

I'll be honest about where this really comes from. The longer I work in IT, the more reluctant I've become to automate things that have the potential to break quietly.

Automation is wonderful right up until it isn't. The script that's hummed along untouched for two years quietly stops firing after an editor update, or a renamed command, or a path that moved — and you don't find out, because the whole appeal was that you'd stopped thinking about it. It fails silently, and it bills you later, in a debugging session you didn't budget for, usually at the worst possible moment.

A manual ritual fails *loudly and legibly*. If my Stream Deck button stops working, I notice immediately, because the colour doesn't change when I press it, and I can fix it with my eyes and a few seconds. There's no hidden machinery to spelunk through. The whole system fits in my head.

So no — I'm not going to automate the colour of my editor. I'm going to keep pressing the button, on purpose, because the pressing is the part that works.