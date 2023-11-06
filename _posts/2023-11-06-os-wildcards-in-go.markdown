---
layout: post
title:  "OS Wildcards in Go"
date:   2023-11-06 07:11:42 +0000
categories: programming go
---

I'm relatively new to Go, in spite of having to be a coder for most of my 30+ year career.  I've programmed in everything from assembler, 
through C, C++, C#, and some obscure languages (such as RTL/2).  Most recently I've found Python to be a language that works well for me - especially
for the ad-hoc nature of the kind of things I need to create these days.  However, at [Mattermost](https://www.mattermost.com), we use Go
for our server platform and the fact that I can share a binary with a customer, rather than needing them to install all kinds of pre-reqs to
run a shell script or a Python script is becoming more and more valuable to me.

Last week, I needed to crate a utility that would help us [pull a support packet](https://github.com/jlandells/mm-packet-pull) from a system
that was down.  This should have been a relatively trivial piece of code to write, as I was mostly collecting and copying files to a single 
location in order to prepare a compressed file.  

I had the code *mostly* working, making use of the Go standard library [os/exec](https://pkg.go.dev/os/exec@go1.21.3) which gives us a fairly
simple way to call the underlying OS.  It was working everywhere, apart from the line where I was trying to copy the contents of a log directory.

This was my code:

```go
cmd := exec.Command("cp", logDirectory+"/*", targetDirectory)
```

Everywhere else that I used a similar construct, it worked perfectly, but `cp` was helpfully returning nothing but `Error 1`!

I spent way too long trying to figure out what was going on, or trying to better understand why the `cp` command was failing, but then it
hit me: This was the only time I was using a wildcard!

You see, in most cases, wildcard expansion is handled by the shell - not by the command itself.  I was expecting this to kind of happen
"by magic".  I expected Go to just handle it for me, but I figured out that by not executing inside a shell, it was simply failing!  Since my
code will only be running on Linux, I was able to convert my code to this:

```go
copyCommand := fmt.Sprintf("cp %s/* %s", logDirectory, targetDirectory)
cmd := exec.Command("sh", "-c", copyCommand)
```

This works because the copy command is now running inside a shell (`sh -c` starts a shell and runs a command inside it), meaning that the
wildcards are automatically expanded for me.  It has the added benefit of providing the complete copy command in a handy variable (`copyCommand`)
which I can print out in its entirety for debugging.

This same technique is also useful if you want to chain commands together, such as `netstat -tulnp | grep 443`.  Again, this kind of struct
really needs a shell to work correctly but in this case, will not fail when you run it, as the `netstat` will complete correctly - it's just that the
`grep` will be ignored!

**Question**: Given this aproach to calling OS commands to run inside a shell, would you handle *all* OS calls in this manner (meaning you can easily write a 
single function to call them), or would you keep it in reserve to use only when needed?  

Let me know what you think!