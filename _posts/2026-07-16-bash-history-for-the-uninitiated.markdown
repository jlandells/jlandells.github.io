---
layout: post
title: "Bash History for the Uninitiated"
date: 2026-07-16 09:35:00 +0100
categories: shell
---

There's a particular sight I've never quite got used to, even after all these years around Linux. You're pairing with someone thoroughly competent — someone who knows their way around a system better than most — and they need a command they ran about ninety seconds ago. So they type the whole thing out again. Character by character. A forty-character `docker` invocation, or a `tar` line with three flags and two paths, reconstructed from memory, typo and all.

The shell remembered that command. It's sitting right there, a keystroke away. Nobody ever told them.

What fascinates me is that none of this is new. Bash's history features have been quietly sitting in the shell for decades, waiting to be noticed. They aren't hidden and they aren't clever — they're just undiscovered, and discovery, not difficulty, is the entire barrier. So this post is my attempt to lower it: not a tour of every trick, just the handful that actually change your day.

A confession first, because I'm not writing this from a lofty perch. Most of what follows I've leaned on for years — I've been running `history` since roughly the last millennium. But reverse-search, the one feature below I'd now least want to give up, I only came to relatively recently, and for a daft reason: I'd got it into my head that it was some non-standard extra that needed a fiddly bit of setup before it would work. It isn't, and it doesn't. It's built in, it's one key, and it had been sitting there the whole time. So if any of this reads as "how did I not know that", you're in good company — the company includes me.

## The gateway drug: the up-arrow

Everyone finds this one eventually. Press the up-arrow and the previous command reappears; press it again for the one before that; down-arrow walks back towards the present. It's the first thing people discover, and for one or two commands back it's genuinely the right tool — quick, obvious, no thinking required.

The trouble is that it's the *only* thing most people discover, and so it gets pressed into service for jobs it's hopeless at. I have watched someone who knew, for a fact, that they'd run a particular command that morning, press the up-arrow again and again — with no idea whether it was ten commands back or fifty, just waiting for it to reappear — rather than simply retyping the eight characters. The up-arrow is a lovely on-ramp. It is not a search tool. Almost everything below exists because the up-arrow runs out of road so quickly.

## Then someone shows them `history`

The next revelation is usually the `history` command, which prints your recent commands with a number against each one:

```
 380  git pull
 381  make
 382  docker compose up
 383  terraform apply
```

On its own, `history` will happily print the whole lot, which is seldom what you're after — so give it a number: `history 20` lists just the last twenty.

> A caveat for anyone on a Mac, because it trips people up constantly: the above is *bash* talking. Since Catalina, macOS has defaulted to zsh rather than bash, and in zsh `history` is wired to the `fc` builtin, which lists only the last sixteen entries by default — to see everything there you'd type `history 1`. Same command name, a different shell underneath, noticeably different behaviour. If a short list is all you've ever seen, that's almost certainly why.

And once every command carries a number, you can re-run one directly: `!383` fires off `terraform apply` again without retyping a character.

The number really earns its keep piped into `grep`. For years, before reverse-search and I were properly acquainted, this was exactly how I dug out an older command:

```
$ history 100 | grep vi
```

The last hundred entries, filtered down to the ones mentioning `vi`, so I could pick out the number I wanted and run it by hand. It works perfectly well and I got a great deal of mileage out of it — and it's precisely the sort of nonsense the final section below does away with entirely.

And here's where I have to be honest about the comedy of it, because I've seen this play out more than once. The moment someone learns `history` and `!383`, the up-arrow goes straight out of the window — and gets replaced by something *slower*. To fetch the previous command, they'll now type `history`, scan a wall of output for the number, and then type `!382`. Three steps and a visual search to do what one tap of the up-arrow did a moment ago. The tools are supposed to compound; instead each new one shoves the last aside. Worth remembering that the right shortcut depends entirely on where the command is: last one, use the up-arrow; three-hundred back, `history` earns its keep.

## The `!` toolkit

Once you're comfortable with `!383`, the rest of the history-expansion family is worth a little of your time, because a few of them come up constantly.

`!!` is simply "the previous command". On its own it's no better than the up-arrow, but its real use is building on what you just ran. The classic is forgetting `sudo`:

```
$ apt update
Permission denied
$ sudo !!
```

That second line expands to `sudo apt update`. It's probably the single most-used history shortcut there is.  You might have seen someone use it, but didn't quite catch how they did it, and when you asked, you got that condescending response of, "oh, you don't know this shortcut?", rather than an explanation!

`!$` is "the last argument of the previous command", which sounds niche until you notice how often you act on the same file twice:

```
$ mkdir /opt/app/releases
$ cd !$
```

`cd !$` becomes `cd /opt/app/releases`. Make a thing, then move into it, without naming it twice.

And two ways to reach further back by content rather than by counting:

- `!string` re-runs the most recent command that *starts* with that string. `!git` grabs your last `git` command; `!terra` your last `terraform` one.
- `!-2` runs the command two back, `!-3` three back, and so on — relative counting when you can see it on screen but can't be bothered with the absolute number.

**One important safety note**: Most of these fire *immediately* — `!git` doesn't show you what it found and wait for a nod; it runs it. If you misremember and your last `git` command was a `git reset --hard`, that's an unwelcome surprise. If that makes you nervous (it should, a little), add this to your `~/.bashrc`:

```bash
shopt -s histverify
```

With `histverify` set, a history expansion is placed on the command line for you to look at and press Enter to run, rather than being executed the instant you hit return. You get the speed of `!` with a moment to check you got what you meant. It's the sort of small, considerate safeguard that costs nothing and saves the occasional bad afternoon.

## The one to actually learn: `Ctrl-r`

If you take one thing from this post, take this one. `Ctrl-r` is reverse incremental search, and it's the feature that made every trick above feel like a museum piece for me.

Press `Ctrl-r` and start typing any fragment of a command you ran before — it doesn't have to be the start, any part of it will do. The shell shows you the most recent match as you type:

```
(reverse-i-search)`comp': docker compose up
```

Three characters and there's the command. From here:

- **Enter** runs it.
- Press **`Ctrl-r` again** to step back to the *next* older match — keep going until the one you want appears.
- Press an **arrow key** (or `Ctrl-e`) to drop the match onto the command line for editing rather than running it — perfect when you want last time's command with one path changed.
- Press **`Ctrl-g`** to bail out entirely and get your empty prompt back.

That's the whole interface, and it replaces almost all the counting and scanning above. No numbers, no walls of output, no pressing a key forty times — just a fragment of what you remember and it surfaces the command. I use it more than any other shortcut in this list, dozens of times a day, and I really notice when I'm on a box where that shortcut isn't there.

> (There's a forward-search sibling on `Ctrl-s`, but be warned: on many terminals `Ctrl-s` is swallowed by ancient flow-control and appears to freeze your session — `Ctrl-q` un-freezes it. `Ctrl-r` is the one to build the habit around.)

## What's worth internalising

The temptation with a post like this is to treat it as a list to memorise, and that's exactly the wrong lesson — it's how you end up with someone typing `history` to fetch the previous command. You don't need forty shortcuts. You need two or three so worn-in that your fingers reach for them without asking: the up-arrow for the last command or two, `Ctrl-r` for anything older, and `!$` for acting twice on the same thing. Everything else is worth *knowing exists* so you can look it up the day you need it, and no more than that.

And if you work alongside people who still retype their morning's commands from memory — most of us do — this is one of the cheapest, most disproportionately appreciated things you can pass on. It took me embarrassingly long to reach for `Ctrl-r` myself, and I'd have been glad of someone tapping me on the shoulder years earlier. Be that someone. It's one key.
