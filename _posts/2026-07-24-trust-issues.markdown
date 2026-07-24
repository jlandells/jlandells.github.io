---
layout: post
title: "Trust Issues"
date: 2026-07-24 08:54:00 +0100
categories: bash scripting
---

Over the last couple of weeks I've written a run of posts about building shell scripts you can trust: meaningful exit codes, defensive parsing, tidying up after yourself. This one is the mirror image. Instead of how to build a good script, it's about how to spot a bad one from across the room — the three tells I run into more than any others.

The word I want is *smell*. If you learned software engineering in a language other than English, or you're simply new to the trade, the term can look an odd one, so it's worth a paragraph. A *code smell* is a surface detail that hints at a deeper problem underneath. It isn't a bug in itself — it doesn't stop the code compiling or running — it's a whiff that makes an experienced engineer wrinkle their nose and go looking. The term was coined by Kent Beck and popularised in the 1999 book *Refactoring*, which he wrote with Martin Fowler, in a chapter titled "Bad Smells in Code". The metaphor is the household one exactly: a funny smell in the kitchen might be nothing more than a strong cheese, or it might be something rotting behind the units. You don't know until you look. But you always look.

Which brings me to the afternoon that prompted this post. I'd handed a build over to Claude Code and was watching it run. Partway through, a line scrolled past: *Waiting 30 seconds for Docker to come up.* Thirty seconds. Not "wait until Docker is up" — just thirty seconds, hand on heart, and hope for the best. And sure enough, a phase further down failed outright, and the build carried on regardless, cheerfully working through the remaining steps on top of a foundation that had already collapsed. That was enough to make me open the scripts and actually read them. They were littered with `exit 1`.

Three smells, one build. Here they are in turn.

## `exit 1` for everything

I've written about this one at length already, so I'll keep it brief here and point you at [the full post]({{ site.url }}/shell/2026-07-13-shell-script-exit-codes.html). The smell is a script that answers `1` to every question it's ever asked. Something went wrong — but what? A missing file, a permission error and a failed network call all leave by the same door wearing the same number. The exit code is the one thing a script can tell a calling process about *how* it failed, and `exit 1` everywhere throws that away, reporting only *that* it failed.

The fix is cheap: give each distinct failure its own code, and write them down in a comment block at the top so the next person — or the next script up the chain — can tell them apart. That's the whole of the earlier post. The reason it leads here is that it's the smell I meet most often, and, as of this month, the one I've now caught an LLM committing more than once.

## Thirty seconds and a prayer

This is the smell that irritates me the most - there's no excuse for it!

`sleep 30` — a fixed wait for something to become ready — is a guess wearing the costume of a safeguard. Pick the number too low and you march on before the thing is up, which is how you get a deployment building on a service that isn't there yet. Pick it too high and every run pays the full tax whether it needs to or not; a service that was ready in three seconds still costs you thirty. And, as my Docker example showed, the wait often gates nothing at all — it pauses, tells you nothing about whether the thing actually came up, and then the script blunders on either way. It's the worst of both worlds: a delay that provides false reassurance and no protection.

The honest version doesn't guess. It *asks*, repeatedly, until it gets a yes or runs out of patience:

```bash
#!/bin/bash
#
# wait-for-service.sh — block until a service is genuinely ready, or give up.
#
# Exit codes:
#   0  service became ready within the budget
#   1  service did not become ready in time
#
set -Eeuo pipefail

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# How hard to try before giving up. MAX_ATTEMPTS * INTERVAL is your
# worst-case wait, so these two numbers ARE your timeout — choose them on
# purpose. Here: up to 30 attempts, 2s apart, so ~60s at most.
readonly MAX_ATTEMPTS=30
readonly INTERVAL=2

# The readiness test. It must ask the REAL question — "can the service do
# its job?" — not the cheap one. A TCP port being open, or a container
# showing as "running", is not the same as the thing behind it being ready
# to serve. Point this at a health endpoint, a real query, whatever proves
# it's genuinely up.
is_ready() {
  curl --silent --fail --output /dev/null "http://localhost:8080/healthz"
}

attempt=1
while (( attempt <= MAX_ATTEMPTS )); do
  if is_ready; then
    log "service ready after ${attempt} attempt(s)"
    exit 0
  fi
  log "not ready yet (attempt ${attempt}/${MAX_ATTEMPTS}); waiting ${INTERVAL}s"
  sleep "${INTERVAL}"
  attempt=$(( attempt + 1 ))
done

# Fell out of the loop having never succeeded. THIS is the line the naive
# 'sleep 30' throws away: an explicit, honest failure with a code that says
# so, instead of ploughing on and hoping.
log "[error] service not ready after $(( MAX_ATTEMPTS * INTERVAL ))s; giving up"
exit 1
```

Three things in there are worth calling out, as they're where I see people get it half-right.

The first is that the two numbers at the top *are* your timeout. Attempts times interval is your worst-case wait, so pick them deliberately rather than leaving a bare `sleep 30` to stand in for a decision you never actually made. Sixty seconds of two-second polls is a very different promise from thirty seconds of nothing, and this way the promise is written down.

The second is what you test, and it matters more than the loop around it. It is tempting to check the cheap thing — is the port open, is the container "running" — because it's easy. But a port answering is not the same as the service behind it being ready to serve, and a container marked running may still be mid-startup. Test the real question: hit a health endpoint, run a query that only succeeds once the thing is actually up. Poll the wrong signal and you've built a loop that waits patiently for the wrong answer.

The third is the part people leave out most often, and it's the whole point: the loop is worthless without a verdict at the end. I've lost count of the scripts that poll nicely, run out of attempts, and then simply *carry on* — no better off than the `sleep`, because they never checked whether the polling actually succeeded. That final `exit 1` after the loop is the entire difference between "wait, and if it never comes up, stop and say so" and "wait a bit, then proceed regardless". The first is a readiness check. The second is `sleep 30` with extra steps.

> **Note:** I'm always banging on about not just blindly calling `exit 1`.  In this case though, it's the only non-zero exit in the script, and is clearly documented in the comments at the start.

A word on making the interval cleverer, since it always comes up. You can have the wait *back off* — grow the gap between attempts — and for something hammering a remote API that's often the polite, sensible thing. For a local readiness check I'd usually not bother: a fixed, short interval is simpler and you'll not out-clever it. And be especially wary of *exponential* backoff here. Double the gap a few times and you're soon sleeping thirty-two seconds at a stretch, which means a service that came up two seconds into that gap sits there ready and ignored while you wait out the rest — and you can sail past any sensible deadline entirely. If you do back off, cap the interval. Unbounded doubling is its own smell.

## Making the red go away

The third smell is the quietest, which is exactly what makes it dangerous. It's the reflex to *silence* a failure rather than handle it. Three tells, in rough order of how often I meet them:

- `some_command || true` — "run this, and if it fails, never mind." Occasionally that's a legitimate, reasoned choice. Far more often it's there because the command was failing and the `|| true` made the noise stop, which is not remotely the same as making the problem stop.
- `some_command 2>/dev/null` — the error message was inconvenient, so it's been routed to the bin. The command can still fail; you've simply blindfolded yourself to why.
- No `set -e` at all — the whole script runs on regardless of what fails inside it, each step blithely assuming the last one worked.

All three produce the same outcome: a script that fails somewhere in the middle and reports success at the end. Green ticks the whole way down, and the thing it was meant to do never happened. I'd take a loud, ugly failure over that every time — at least the loud one tells you where to look.

I'll not re-tread the fix, because it's most of an earlier post: turn the safety on with `set -Eeuo pipefail`, let failures be loud, and suppress a specific error only when you have a specific, stated reason for doing so. [Bash Scripts You Can Come Back To]({{ site.url }}/bash/scripting/2026-07-14-bash-scripts-you-can-come-back-to.html) walks through it properly — including why `pipefail` matters, and the places where `set -e` quietly doesn't save you.

This is the one I've been seeing most from LLMs lately. Ask for a script that does a job and there's a fair chance a `|| true` or a redirected `stderr` arrives bundled with it — not out of any malice, but because the model has read an ocean of code that does exactly that, and it's handing you the average of what it learned.

## Trust issues

Line the three up and they rhyme. `exit 1` fails without telling you how. `sleep 30` assumes success will simply turn up if it waits long enough. `|| true` hides the failure altogether. Three different ways for a script to be dishonest about its own state — and once you can't trust what a script tells you about whether it worked, you can't trust anything downstream of it either. Hence the title.

Now the part I actually want you to take away. I met all three of these in a single build, in code generated by an LLM. I use these tools daily and rate them highly, so this isn't a complaint about the technology; it's an observation about what the technology *is*. A model gives you the average of the code it was trained on, and the world's shell scripts are, in aggregate, riddled with exactly these smells. So it reproduces them faithfully — and it does so wrapped in output that reads as fluent, confident and thoroughly professional. That's the trap. The exit codes post caught one anti-pattern and I half-treated it as a one-off. It isn't! It's a pattern of patterns, and the better the surrounding code looks, the *more* careful the review needs to be, not less — because a broken core dressed in tidy, well-commented prose is harder to distrust, not easier.

And before anyone reaches for the obvious answer: no, a line in your `CLAUDE.md` telling it not to do these things does not get you off the hook. I know this because mine forbids all three by name. My standing instructions could not be plainer on the subject. And yet here I am, having spent an afternoon pulling `exit 1` out of thirty-odd scripts that a very well-behaved assistant assured me it had written to my standards. An instruction is a request, not a guarantee. The code review is the only thing that actually checks the work — and that stays true no matter whose name is on the commit.

## Related Posts

- [Shell Script Exit Codes]({{ site.url }}/shell/2026-07-13-shell-script-exit-codes.html)
- [Bash Scripts You Can Come Back To]({{ site.url }}/bash/scripting/2026-07-14-bash-scripts-you-can-come-back-to.html)
- [Well-Behaved Bash Scripts]({{ site.url }}/bash/scripting/2026-07-14-well-behaved-bash-scripts.html)
