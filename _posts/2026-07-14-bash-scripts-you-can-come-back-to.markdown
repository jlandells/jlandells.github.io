---
layout: post
title: "Bash Scripts You Can Come Back To"
date: 2026-07-14 10:30:00 +0100
categories: bash scripting
---

Yesterday I wrote about exit codes: the number a script hands back to say whether it worked, and the quiet damage done by scripts that answer `1` to every question they're ever asked ([the previous post is here]({{ site.url }}/shell/2026-07-13-shell-script-exit-codes.html)). That post was about how a script *reports* failure once it has already happened. This one is about the other half of the same job — not failing blindly in the first place, and leaving behind something a human can actually read.

I have a simple test for a script. Could I come back to it in two years, having forgotten everything about it, and be productive again inside a couple of minutes — from the comments, the usage message and the structure alone? Most scripts fail that test, and they fail it in the same few ways every time.

I see a lot of shell scripts, and a depressing number of them are held together with optimism. Positional parameters that fall over the moment someone forgets an argument — or, worse, the moment an argument arrives blank from a calling script. No error handling worth the name. And this one still catches me out: senior Linux engineers, people who should know better, who have never met `set -euo pipefail`, let alone a trap handler.

None of the fixes are difficult. It's a small set of habits, and once they're muscle memory they cost nothing. Here's a script that has them. It doesn't do anything interesting — it pretends to deploy a release — because the scaffolding is the point, not the payload.

```bash
#!/bin/bash
#
# deploy-release.sh — deploy a build artefact to a target environment.
#
# Usage: deploy-release.sh -r RELEASE [-e ENVIRONMENT] [-v] [-h]
#
# Exit codes:
#   0  success
#   2  usage error (unknown option, or an option missing its value)
#   3  no release specified (-r is required)
#   4  unknown environment
#   *  any other non-zero code is passed straight through from a failed
#      command by the ERR handler (see 'Error handling' below)
#

set -Eeuo pipefail

# --- Defaults -------------------------------------------------------------
ENVIRONMENT="staging"
VERBOSE=false

# --- Logging --------------------------------------------------------------
# One helper, so every line is timestamped without having to remember to do
# it. These invariably end up in a logfile, and timing is often the answer.
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# --- Error handling -------------------------------------------------------
# Fires on any uncaught error (see 'set -Eeuo pipefail'). Reports the line
# that actually failed and preserves that command's exit code.
error_handler() {
  local exit_code=$?
  local line=${BASH_LINENO[0]}
  log "[error] ${0##*/} failed at line ${line} (exit code ${exit_code})"
  exit "${exit_code}"
}
trap error_handler ERR

# --- Usage ----------------------------------------------------------------
usage() {
  cat << EOF
Usage: ${0##*/} -r RELEASE [-e ENVIRONMENT] [-v] [-h]

Deploy a build artefact to a target environment.

Required:
  -r RELEASE       Release version to deploy, e.g. 2026.07.1

Optional:
  -e ENVIRONMENT   Target: staging or production (default: ${ENVIRONMENT})
  -v               Verbose output
  -h               Show this help and exit
EOF
}

# --- Parse arguments ------------------------------------------------------
while getopts ":r:e:vh" opt; do
  case "${opt}" in
    r)  RELEASE="${OPTARG}" ;;
    e)  ENVIRONMENT="${OPTARG}" ;;
    v)  VERBOSE=true ;;
    h)  usage; exit 0 ;;
    \?) log "[error] unknown option: -${OPTARG}"; usage; exit 2 ;;
    :)  log "[error] option -${OPTARG} requires a value"; usage; exit 2 ;;
  esac
done

# --- Validate -------------------------------------------------------------
# A missing -r is the classic blank-parameter failure. Catch it by name,
# here, with a meaningful exit code — don't let it ripple downstream.
if [[ -z "${RELEASE:-}" ]]; then
  log "[error] no release specified (-r is required)"
  exit 3
fi

case "${ENVIRONMENT}" in
  staging|production) ;;
  *) log "[error] unknown environment: ${ENVIRONMENT}"; exit 4 ;;
esac

# --- Main -----------------------------------------------------------------
log "deploying release ${RELEASE} to ${ENVIRONMENT}"
if [[ "${VERBOSE}" == true ]]; then
  log "verbose mode enabled"
fi

# ... real deployment steps would go here ...

log "done"
```

Nothing in there is clever, and that's deliberate — clever is the enemy of the two-year test. Let me take it apart, because every section is earning its place.

## Named arguments, and a usage() you'll actually read

Positional parameters are where most scripts begin to rot. `$1`, `$2`, `$3`, threaded through the body with no clue as to what each one means. Two failure modes follow, as night follows day. Forget an argument and everything shifts up by one, so `$2` is now holding what you meant for `$3`, and the script runs on regardless. Or — the nastier one — a parameter arrives blank because it came from another script that didn't set it. An empty string is a perfectly legal value as far as the shell is concerned, so nothing complains. The script simply does the wrong thing with nothing, quietly, and you find out later.

Named arguments through `getopts` deal with both. Compare these two invocations:

```
deploy-release.sh 2026.07.1 production 1
deploy-release.sh -r 2026.07.1 -e production -v
```

The second reads like a sentence. The first is a riddle, and that trailing `1` is anybody's guess until you go and read the source. Named arguments also mean order stops mattering and missing values become detectable, because you're no longer relying on position to carry meaning.

The `usage()` function is the contract. It's the first thing future-you reads, and if it's honest and complete you may not need to read anything else. Mine lives right next to the parser so the two can't drift apart, and it's reachable with `-h` so the script can explain itself on demand.

And here's something I only recently learned, after more years of shell scripting than I care to admit to. Look closely at the `getopts` string:

```bash
while getopts ":r:e:vh" opt; do
```

The colons *after* the letters are the ones everyone knows: `r:` and `e:` mean "this option takes a value." Nothing new there. But that colon at the very *start* — `":r:..."` — is a different thing entirely, and I'd never once seen it in all my years of reading other people's scripts.

A leading colon puts `getopts` into what the manual calls "silent error reporting". The name is misleading: nothing is being ignored. It means `getopts` keeps its own mouth shut when it meets a bad option, and hands *you* the offending character in `OPTARG` so that your own `\?)` and `:)` cases can do the reporting instead. Without it, `getopts` prints its own terse message — `illegal option -- z` — in its own voice, and leaves `OPTARG` unset.

That last detail matters more than it looks, because of the next section. With `set -u` switched on, an unset `OPTARG` is a fatal "unbound variable" error the instant your handler touches it. So the default mode doesn't just rob you of a decent error message; under `set -u` it actively blows up in the one place you were trying to be careful. The leading colon lets you own the message, own the exit code, and stay out of that trap.

## set -Eeuo pipefail — one line, four decisions

This is the line the post is really about, and it's four separate decisions bundled into one incantation. Each is worth understanding rather than pasting.

`-e` (errexit): exit the moment any command fails, instead of blundering on. This is the big one, and it's also the one people trust too much — more on that in a moment.

`-u` (nounset): treat any reference to an unset variable as an error. This is what catches the blank-parameter class of bug before it can do harm. It's also why you'll see `"${RELEASE:-}"` in the validation check — the `:-` supplies an empty default so that *testing* for the variable doesn't itself trip `-u`. When you genuinely mean "this might be unset", you say so explicitly.

`-o pipefail`: make a pipeline fail if *any* stage fails, not just the last one. Without it, `false | tee log.txt` reports success, because `tee` succeeded and `tee` had the last word. That is precisely the sort of lie that costs you an afternoon.

`-E` (errtrace): make the `ERR` trap reach inside shell functions (and subshells, and command substitutions). This one is subtle, and it's the reason you can go years without knowing you're missing it. Leave `-E` off and `set -e` still stops the script when a command fails inside a function — the safety is intact. What you lose is the handler: your `ERR` trap fires only for errors at the top level, not for those inside functions, so the script still dies but you get errexit's bare exit instead of your timestamped, line-numbered message. If your errors tend to happen at the top level, or you've no `ERR` trap at all, you'll never notice the gap. But once you've gone to the trouble of writing a handler, `-E` is what makes it trustworthy across the whole script rather than just its outermost layer.

Now, the honesty. `set -e` is a safety net with real holes in it, and you should know where they are rather than assume it has you covered:

- It does **not** fire for a command whose result is being tested — anything in an `if` or `while` condition, either side of `&&` or `||` (bar the last), or after a `!`. A `grep` that "fails" inside an `if` is being asked a question, not reporting a fault, so the script carries on. That's correct behaviour, but it surprises people.
- It is **suspended entirely** inside a function that is itself called in a tested context. Call `do_work` on its own and a failure inside it aborts as you'd expect; call `if do_work; then ...` and `set -e` goes to sleep for the whole function. That holds for a function used in any tested context — an `if`, a `while`, either side of `&&` or `||` — and the difference is exactly that stark.

The lesson isn't "don't use `set -e`" — you absolutely should. It's that `set -e` catches the failures you didn't anticipate, and it can't be your entire strategy for the ones you did. Which brings us to the handler and the explicit checks.

## The error handler — knowing where the body is buried

The trap is the safety net:

```bash
error_handler() {
  local exit_code=$?
  local line=${BASH_LINENO[0]}
  log "[error] ${0##*/} failed at line ${line} (exit code ${exit_code})"
  exit "${exit_code}"
}
trap error_handler ERR
```

A single `trap ... ERR` catches everything `set -e` throws, which sounds tidy right up until you realise you've traded per-command detail for one catch-all. So the one thing the handler absolutely must tell you is *where* it died. There are two traps for the unwary in getting that right, and I fell into both before I got here.

The first: don't read `$LINENO` inside the handler. `$LINENO` means "the line running right now", and once you're inside the function, the line running right now is a line *in the function*. It'll cheerfully report the line number of the `log` call in the handler itself, the same wrong answer every single time. Use `${BASH_LINENO[0]}` instead — that's the line in the caller that triggered the trap, which is the line you actually want.

The second is the callback to yesterday's post, and it's the one I'm most keen for you to notice. A naive handler ends with `exit 1`. Do that and every uncaught failure, whatever its real exit code, gets flattened to `1` on the way out the door — which is exactly the anti-pattern the exit codes post was about, committed inside the very error handler meant to be doing things properly. The fix is two lines. Capture `$?` as the *first* statement in the handler, before any other command overwrites it, and re-exit with that value. A `grep` that failed with code 2 now leaves the script with code 2, and the caller upstream can tell "file wasn't there" from "permission denied".

## The safety net is not the plan

The trap is for the failures you didn't see coming. For the ones you can see coming, don't lean on the safety net — check them where they happen, say something useful, and exit with a code you chose on purpose:

```bash
if [[ -z "${RELEASE:-}" ]]; then
  log "[error] no release specified (-r is required)"
  exit 3
fi

case "${ENVIRONMENT}" in
  staging|production) ;;
  *) log "[error] unknown environment: ${ENVIRONMENT}"; exit 4 ;;
esac
```

A bad option is `2`, a missing release is `3`, an unknown environment is `4`. They're deliberate and they're distinct, so anything calling this script can tell the difference between the ways it can refuse. That's the whole lesson from yesterday, put to work: meaningful exit codes aren't an afterthought bolted on at the end, they're how a defensive script talks to whatever is calling it.

## Every line wears a timestamp

Every line the script emits goes through `log()`, and `log()` stamps it with the date and time. This is not decoration. These scripts almost always end up writing to a logfile, and when something has gone wrong at some ungodly hour, the *timing* of events is frequently the thing that cracks the case — which step hung, how long the slow bit took, whether two things collided. A logfile of untimed lines throws all of that away. Putting the timestamp in one helper rather than typing `$(date ...)` forty times also means you can't forget it and you can't let it drift out of format.

## The two-year test

Add it up and none of it is exotic: named arguments, a real usage message, four sensible shell options, a handler that tells the truth about where it failed, explicit checks with codes that mean something, and a timestamp on everything. The `# --- section ---` banners aren't there to look pretty either — they're so that two-years-from-now-you can scroll the file and find the parsing, the validation or the main logic without reading a line of it.

That's the actual point of all of this. Not correctness for its own sake, and certainly not cleverness — but kindness to the person who has to pick this up cold and get it working under pressure. That person is almost always you, and you will have forgotten everything.

And even this script has further to go — there's no cleanup on exit, no dry-run mode, no locking to stop two copies running at once. But it fails safely, it fails legibly, and it tells you where and why. That's the floor, not the ceiling, and it's a floor an alarming number of production scripts never reach.


## Related Posts

- [Shell Script Exit Codes]({{ site.url }}/shell/2026-07-13-shell-script-exit-codes.html)
- [Well-Behaved Bash Scripts]({{ site.url }}/bash/scripting/2026-07-14-well-behaved-bash-scripts.html)