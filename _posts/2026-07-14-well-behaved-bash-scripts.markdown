---
layout: post
title: "Well-Behaved Bash Scripts"
date: 2026-07-14 20:00:00 +0100
categories: bash scripting
---

I ended the last post ([Bash Scripts You Can Come Back To]({{ site.url }}/bash/scripting/2026-07-14-bash-scripts-you-can-come-back-to.html)) with a confession. The tidy little `deploy-release.sh` I'd spent the whole piece building still had three holes in it: it tidied up nothing when it finished, it had no way to show you what it *would* do before committing, and nothing stopped two copies trampling over each other. I called that the floor, not the ceiling.

This post is about the next few steps up. The three gaps share a theme — each is about a script being considerate, both to the machine it runs on and to the person at the keyboard. Leftover scratch files, a destructive command you couldn't preview, two deployments racing each other into the same directory: none of these are correctness bugs exactly, but they're all a script behaving like an inconsiderate house guest. So here's the same script again, taught some manners.

```bash
#!/bin/bash
#
# deploy-release.sh — deploy a build artefact to a target environment.
#
# Usage: deploy-release.sh -r RELEASE [-e ENVIRONMENT] [-n] [-v] [-h]
#
# Exit codes:
#   0  success
#   2  usage error (unknown option, or an option missing its value)
#   3  no release specified (-r is required)
#   4  unknown environment
#   5  another instance is already running (could not acquire the lock)
#   *  any other non-zero code is passed straight through from a failed
#      command by the ERR handler.
#
# Note: the locking below relies on flock(1), which is part of util-linux.
# It is present on Linux but NOT on macOS by default — this script is
# written for Linux.
#

set -Eeuo pipefail

# --- Defaults -------------------------------------------------------------
ENVIRONMENT="staging"
VERBOSE=false
DRY_RUN=false
LOCKFILE="/var/lock/deploy-release.lock"

# --- Logging --------------------------------------------------------------
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# --- Run a command (or, in dry-run mode, just say what it would be) -------
# Wrap every command that CHANGES something in run(). In dry-run mode it
# narrates the command instead of executing it; with -v it announces each
# command as it runs; otherwise it runs it silently. Important limitation:
# run() executes a simple command via "$@", so it does NOT understand
# pipes, redirections or shell expansions. Keep those out of anything you
# hand to it, or they won't do what you expect.
run() {
  if [[ "${DRY_RUN}" == true ]]; then
    log "[dry-run] would run: $*"
    return 0
  fi
  if [[ "${VERBOSE}" == true ]]; then
    log "running: $*"
  fi
  "$@"
}

# --- Cleanup on exit ------------------------------------------------------
# Fires on EVERY exit: success, a handled error, or errexit pulling the
# plug. It must therefore be safe to run when there is nothing to clean —
# it will sometimes fire before WORKDIR has even been created, hence the
# guard. Note what is NOT here: we never release the flock. We don't have
# to. See the locking section for why.
cleanup() {
  if [[ -n "${WORKDIR:-}" && -d "${WORKDIR}" ]]; then
    rm -rf "${WORKDIR}"
  fi
}
trap cleanup EXIT

# --- Error handling -------------------------------------------------------
# Reports the line that actually failed and preserves that command's exit
# code (see the previous two posts for why that matters).
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
Usage: ${0##*/} -r RELEASE [-e ENVIRONMENT] [-n] [-v] [-h]

Deploy a build artefact to a target environment.

Required:
  -r RELEASE       Release version to deploy, e.g. 2026.07.1

Optional:
  -e ENVIRONMENT   Target: staging or production (default: ${ENVIRONMENT})
  -n               Dry run: print what would happen, change nothing
  -v               Verbose output
  -h               Show this help and exit
EOF
}

# --- Parse arguments ------------------------------------------------------
while getopts ":r:e:nvh" opt; do
  case "${opt}" in
    r)  RELEASE="${OPTARG}" ;;
    e)  ENVIRONMENT="${OPTARG}" ;;
    n)  DRY_RUN=true ;;
    v)  VERBOSE=true ;;
    h)  usage; exit 0 ;;
    \?) log "[error] unknown option: -${OPTARG}"; usage; exit 2 ;;
    :)  log "[error] option -${OPTARG} requires a value"; usage; exit 2 ;;
  esac
done

# --- Validate -------------------------------------------------------------
if [[ -z "${RELEASE:-}" ]]; then
  log "[error] no release specified (-r is required)"
  exit 3
fi

case "${ENVIRONMENT}" in
  staging|production) ;;
  *) log "[error] unknown environment: ${ENVIRONMENT}"; exit 4 ;;
esac

# --- Acquire the lock -----------------------------------------------------
# Open the lock file on file descriptor 200 (a high number, chosen to stay
# well clear of stdin/stdout/stderr at 0/1/2). The descriptor stays open
# for the life of the script, and the lock lives on that descriptor — so
# the kernel drops it automatically when the script exits and the fd
# closes. That is why cleanup() doesn't touch it: there is nothing to
# release by hand, and no stale lock file left behind.
#
# 'flock -n' means "fail immediately if someone else holds the lock"
# rather than the default, which is to wait. And because the flock sits in
# an 'if' test, set -e stands aside for it (see the last post) — so we get
# to handle the failure ourselves, with a meaningful exit code, rather
# than the script dying with errexit's bare message.
exec 200>"${LOCKFILE}"
if ! flock -n 200; then
  log "[error] another deploy is already running (lock: ${LOCKFILE})"
  exit 5
fi

# --- Set up a working directory -------------------------------------------
WORKDIR="$(mktemp -d)"
log "using working directory ${WORKDIR}"

# --- Main -----------------------------------------------------------------
log "deploying release ${RELEASE} to ${ENVIRONMENT}"
if [[ "${DRY_RUN}" == true ]]; then
  log "dry-run mode: no changes will be made"
fi

run cp "releases/${RELEASE}.tar.gz" "${WORKDIR}/"
run tar -xzf "${WORKDIR}/${RELEASE}.tar.gz" -C "${WORKDIR}"
run systemctl restart "app-${ENVIRONMENT}"

log "done"
```

As before, the payload is fake — it pretends to unpack a release and restart a service — because the scaffolding is the lesson, not the deployment. Let me take the three new pieces in turn.

## Tidying up: the EXIT trap

The last post's `ERR` trap has a calmer sibling, and here it is:

```bash
cleanup() {
  if [[ -n "${WORKDIR:-}" && -d "${WORKDIR}" ]]; then
    rm -rf "${WORKDIR}"
  fi
}
trap cleanup EXIT
```

Where `ERR` fires only when something goes wrong, `EXIT` fires whenever the script leaves *at all* — a clean finish, a handled error, or `set -e` pulling the plug halfway through. That's exactly what you want for cleanup: whatever scratch files, temporary directories or half-finished state the script created, `EXIT` is the one place guaranteed to run on the way out, so it's the one place you can be sure the tidying happens.

The two traps coexist without any special handling. When a command fails, `ERR` fires first — logs the line and re-exits with the real code — and then, because the script is now exiting, `EXIT` fires and the cleanup runs. On a clean finish, `ERR` never fires and `EXIT` runs on its own. You don't have to coordinate them; the shell does it.

The one thing the cleanup function *must* be is safe to run when there's nothing to clean. Look at when the trap is armed — right near the top, long before `WORKDIR` exists. If the script falls over during argument validation, `EXIT` still fires, and `cleanup()` runs at a point when `WORKDIR` is an unset variable. Under `set -u` that's a fatal error inside your own cleanup handler, which is a genuinely maddening thing to debug. Hence the guard: `[[ -n "${WORKDIR:-}" && -d "${WORKDIR}" ]]` checks both that the variable is set (with the `:-` giving `set -u` an empty default to chew on) and that the directory is actually there before `rm -rf` goes anywhere near it. A cleanup handler that can itself blow up isn't a cleanup handler; it's a second bug lying in wait.

## Look before you leap: a dry-run mode

A deploy script is precisely the sort of thing you want to be able to *rehearse*. Dry-run mode lets it narrate every change it would make without making any of them, and the mechanism is a single small wrapper:

```bash
run() {
  if [[ "${DRY_RUN}" == true ]]; then
    log "[dry-run] would run: $*"
    return 0
  fi
  "$@"
}
```

Then every command that changes something goes through it — `run cp ...`, `run tar ...`, `run systemctl ...` — and the `-n` flag flips `DRY_RUN` to `true`. In normal use `run` just executes its arguments; with `-v` it announces each command as it runs it; in dry-run it logs `would run: cp releases/...` and returns success without touching a thing. That last flag also, at long last, gives the inherited `-v` something real to do — it had been carried along since the earlier posts without quite earning its place. The discipline it asks of you is simple and worth stating out loud in a comment, as I have in the script: *anything that changes state goes through `run`; anything that only reads can be called directly.* Get that split wrong — call `rm` directly by mistake — and your "dry" run quietly deletes something, which rather defeats the point.

There's an honest limitation here, and I'd rather flag it than let you discover it the hard way. `run` executes a *simple command* through `"$@"`. It does not understand pipes, redirections or shell expansions — those are interpreted by the shell before `run` ever sees its arguments. `run foo | bar` doesn't wrap the pipeline; it wraps `foo` and pipes the result to `bar` regardless of dry-run. If a step genuinely needs a pipeline, wrap the whole thing in a small function and `run` that function instead. For the common case — one command that changes one thing — the wrapper earns its keep many times over.

## One at a time: locking with flock

This is the part most people won't have seen before, so it's the part that most needs explaining — both here and, more importantly, *in the script itself*. Here's the mechanism:

```bash
exec 200>"${LOCKFILE}"
if ! flock -n 200; then
  log "[error] another deploy is already running (lock: ${LOCKFILE})"
  exit 5
fi
```

First, the Linux-specific caveat, because it bit me once and it'll bite others: `flock` is part of the `util-linux` package. It's on every mainstream Linux distribution, but it is *not* installed on macOS by default. If you develop on a Mac and deploy to Linux — which describes a great many of us — this works in production and fails on your laptop, and the error won't be obvious. Say so in a comment at the top of the file, as I have, so nobody wastes an afternoon on it.

Now the mechanism, one line at a time, because none of it is self-explanatory:

`exec 200>"${LOCKFILE}"` opens the lock file for writing on **file descriptor 200**. The high number is deliberate — 0, 1 and 2 are stdin, stdout and stderr, and low single digits are easy to collide with, so a big number keeps the lock's descriptor well out of everyone's way. The `>` creates the file if it isn't there (which does mean the script needs write access to `/var/lock`, or wherever you put it — worth a thought for whichever user runs this).

`flock -n 200` takes an exclusive lock on that descriptor. The `-n` is the important flag: it means *fail immediately if the lock is already held*, rather than the default behaviour, which is to wait patiently until it comes free. For a deploy you almost certainly want to fail fast and tell the operator "someone's already doing this", not silently queue up behind them.

And here's the elegant bit, the one that ties this section back to the last: **you never see a matching unlock, and there's no lock file to clean up.** The lock lives on the open descriptor, and the kernel releases it automatically the moment the script exits and descriptor 200 closes — whether the script finished cleanly, hit an error, or was killed. This is genuinely lovely behaviour, and it's the reason the old PID-file approach (write your process ID to a file, delete it when you're done, and pray you never get killed mid-run leaving a stale file that locks everyone out forever) belongs in the past. But — and this is the whole point of the section — it is *completely invisible* from the code. Nothing about `exec 200>...` and `flock -n 200` tells the next reader that the lock releases itself, or why there's no cleanup for it, or what on earth descriptor 200 is. That knowledge lives entirely in your head, until you write it down. So write it down. The comment block above this in the script is longer than the code it explains, and that ratio is correct. This is exactly the kind of terse, clever-looking incantation that the two-year test punishes hardest: it works, it's short, and it is unreadable without a paragraph of context.

One last caveat, and it too deserves a comment if it applies to you: `flock`'s guarantees are *local*. On some network filesystems — NFS and CIFS among them — the underlying locking is limited or may not behave as you'd expect, so don't lean on a lock file sitting on a network share to coordinate deployments across several machines. For a single host, which is the common case, it's rock solid.

## The thread running through all three

Step back and the three additions rhyme. Cleanup on exit is the script tidying up after itself. Dry-run is the script letting you check its intentions before it acts. Locking is the script refusing to trample a copy of itself. Good manners, all of it — consideration for the system and for the human.

But the quieter thread is the comments. The `run()` wrapper needed a note about what it can't do. The cleanup handler needed a note about why it's guarded. The lock needed a whole paragraph about a descriptor number and a release that happens by magic. In every case the *code* is short and the *reason* is not, and the reason is the thing that rots first when it lives only in your memory. The previous post was about a script you can come back to. This one adds the corollary: the cleverer and more compact a technique looks, the more it owes the reader an explanation. `flock` on a file descriptor is four tokens and a decade of Unix behaviour. Leave the four tokens and throw the decade away, and you've written something that works and teaches nobody — least of all future-you, arriving cold at 2am with the pager going off.


## Related Posts

- [Shell Script Exit Codes]({{ site.url }}/shell/2026-07-13-shell-script-exit-codes.html)
- [Bash Script You Can Come Back To]({{ site.url }}/bash/scripting/2026-07-14-bash-scripts-you-can-come-back-to.html)