---
layout: post
title: "Shell Script Exit Codes"
date: 2026-07-13 22:30:00 +0100
categories: shell
---

Even today, with all the fancy languages and automation possibilities, shell scripts have a solid place in the toolkit of any Linux engineer.  They enable us to automate mundane tasks in a simple and clear manner, without the need to worry about runtime environments or compilers.  Used well, they're incredibly powerful.

### Exit Codes

One of the fundamentals of shell scripts is that they all use something known as **exit codes**.  These are simply numbers that are sent to the operating system to indicate whether the script was successful or not.  The standard is that if a script ends normally, either through getting to the end without an error, or explicitly being ended through an `exit 0` command, then it sends a `0` to the OS.  This says, "I have completed successfully".  Alternatively, *any* non-zero value indicates an error of some description.

Inexperienced programmers will often use nothing more than `exit 0` for success, and `exit 1` for a failure, but in anything other than the most basic script, this can be problematic.  You might print a helpful error message, but if you're calling your script from some other system, there's a risk that the output has been suppressed or redirected, so you only see a non-zero exit.

Many years ago, I was asked to review a script that an inexperienced colleague had written for a client.  It was 4,500 lines long, and it kept failing, but nobody could pinpoint why.  All they knew was that it was showing "Exit 1" on the command line.  There wasn't even an error message!  When I reviewed it, I found over 100 calls to `exit 1`!  I worked my way through all of them, adding meaningful error messages (even including helpful debug information in these outputs) and giving each call to `exit` a unique number.  I also collated every one of these exit codes into a comment block at the top of the script, so that anyone looking at it could easily see what a specific code meant.

Ultimately, we replaced the script with something far cleaner, but we were only able to do that because with proper exit codes, we could see what was tripping it up.

### AI-Assisted Scripting

Fast forward to today.  I manage a complex demo deployment environment, which relies on a large number of shell scripts to build a demo instance.  Some of them were written by hand, but recently, I've been embracing the technology and allowing Claude Code to write the scripts.  A colleague had an issue with a deployment, so I checked the logs, and what I saw hit me hard: `Script failed with: Exit 1`.  Checking that script, there was a single `exit 0`, and every other exit was `exit 1`!

It's my fault for not checking, but the rest of the script was so professionally written that it didn't even cross my mind to check something so fundamental!  I often say that AI is like a really keen intern who works exceptionally quickly, deals with the mundane work, and has occasional flashes of brilliance.  In this instance though, Claude Code sadly highlighted why we have to review code more thoroughly!  I ended up finding 35 shell scripts with this same anti-pattern!

Once educated on the matter (or at least, once it was highlighted), Claude Code did a great job of going through all scripts and allocating unique exit codes, as well as documenting them in a comment block at the start.  Since making these changes, we haven't had a failure in the same place, but if we do, we'll have a better chance of tracking it down!

### A Few Gotchas

In theory, you can pass *any number* to `exit` as an exit code, however the OS only uses the lowest 8 bits.  Anything beyond those bits (0-255) is reduced modulo 256.

For example:

```bash
$ bash -c 'exit 256'
$ echo $?
0

$ bash -c 'exit 257'
$ echo $?
1

$ bash -c 'exit 999'
$ echo $?
231
```

There are also a few "reserved" exit codes, that are used for specific meanings by convention:

| Exit Code | Meaning                                                                                     |
| --------- | ------------------------------------------------------------------------------------------- |
|    0      | Success                                                                                     |
|    1      | General error                                                                               |
|    2      | Misuse of shell builtins (Bash convention)                                                  |
|  126      | Command found but not executable                                                            |
|  127      | Command not found                                                                           |
|  128+n    | Process terminated by signal *n* (e.g. `SIGINT` -> 130, `SIGKILL` -> 137, `SIGTERM` -> 143) |

#### Why only 8-bits?

Historically, the Unix `wait()` system call stored the child's termination information in a 16-bit status word, with only 8 bits allocated for the program's exit status. POSIX preserves this behaviour, and shells expose only those 8 bits via `$?`.

So, in practice, **255 is the largest distinct exit code you can reliably communicate to another process**. If you need to return richer information, it's common to:
- write structured output (JSON, text, etc.) to stdout or a file
- use `stderr` for diagnostics
- reserve the exit code simply to indicate success/failure or a broad category of error.


### Related Posts:

- [Bash Script You Can Come Back To]({{ site.url }}/bash/scripting/2026-07-14-bash-scripts-you-can-come-back-to.html)
- [Well-Behaved Bash Scripts]({{ site.url }}/bash/scripting/2026-07-14-well-behaved-bash-scripts.html)