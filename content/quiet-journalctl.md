+++
title = "Solving Non-Existent Problems: A Quiet journalctl and Logs That Still Exist"
date = 2026-06-25
+++

I run most of my long-lived processes as systemd `--user` services. Three things keep me there:

- That tidy little **RAM usage** line systemd prints for each unit.
- Every process a service spawns is **visible together** under one tree.
- And almost for free, **stdout and stderr land in journald**, so logs are there if I want them.

That last point is also where the trouble starts. When I open `journalctl -eb` to look at what the system has been doing, I usually do *not* want to scroll through the chatter of a handful of noisy user services. They drown out the messages I actually came to read.

So this is a post about quieting `journalctl` without throwing the logs away. It's a journey that starts with a real problem and then, in the grand tradition of personal infrastructure, cheerfully invents several more.

## 1. The boring, sensible fixes

systemd already gives you `StandardOutput=` per unit. Pointing it somewhere other than the journal solves the *visibility* problem immediately. The only question is what you trade away.

```ini
StandardOutput=null
```

Quietest possible option. It also means the logs are simply gone the moment you decide you needed them — which, in my experience, is the moment right after you choose `null`.

```ini
StandardOutput=file:%h/.cache/somefile.log
```

Now the logs live in a file. Good enough if you only ever care about the **current or last** run and are happy to lose history, because each start truncates the file. My one nag here is paranoia: what if the process spams hard enough to fill the disk in a single run?

```ini
StandardOutput=file:%t/somefile.log
```

Same idea, but %t expands to the runtime directory ($XDG_RUNTIME_DIR, usually a tmpfs). No disk usage — but a tmpfs is RAM, so now a spammy run can fill it and crash other processes writing there. And the cross-reboot persistence is gone too.

```ini
StandardOutput=append:%h/.cache/somefile.log
```

Like the `file:` case, but it **appends** instead of truncating, so old runs survive. Which brings the disk-filling paranoia right back.

This is a perfectly fine place to stop. The rest of this post is me refusing to stop.

## 2. Non-existent problem #1: "but the disk might fill up"

The fix for unbounded growth is a logger that rotates. `svlogd` (from runit/daemontools-land) reads stdin and writes into a directory, rotating log files and deleting old ones once it hits configured limits. Pipe a service through it and the disk-filling fear is handled for good.

```ini
ExecStartPre=/bin/sh -c "mkdir -p ~/.cache/svlogd/sway"
ExecStart=/bin/sh -c 'sway | svlogd -ttt ~/.cache/svlogd/sway'
```

The `ExecStartPre` is there because `svlogd` won't create its log directory for you. This works, but the shell wrapper introduces two warts:

- The service's main process is now `/bin/sh`, an **extra PID** we didn't want.
- Because of that, **`MAINPID` is the shell**, not `sway`. systemd is now tracking the wrong thing.

For something special like `sway` I don't mind giving it its own log directory. What I *don't* want is to hand-manage a separate directory for every other little service.

## 3. Dropping the shell with `logpipe`

The shell only exists to set up a pipe and then get out of the way — except it doesn't get out of the way, it lingers as the main process. [`logpipe`](https://github.com/navneetankur/logpipe) does exactly the setup and then *exec*s into the real program, so it does not stick around after wiring up the redirection.

```ini
ExecStart=logpipe svlogd -ttt ~/.cache/svlogd/sway -- sway
```

Now `sway` is the main process, `MAINPID` is correct, and the extra PID is gone. The only remaining annoyance is the one I admitted I didn't want to live with: I still have to create and manage **a directory per process**.

## 4. Non-existent problem #2: "one directory per process is too many directories"

So let's make `svlogd` itself a shared service. Run one `svlogd` that reads from a **named pipe (FIFO)** and writes into a single shared directory. Everything else just writes to the pipe.

```ini
ExecStartPre=-/usr/bin/mkfifo -m 600 %t/svlogd.pipe
ExecStartPre=/usr/bin/mkdir -p %h/.cache/svlogd/others
ExecStart=/bin/sh -c 'exec 3<>%t/svlogd.pipe; exec svlogd -ttt %h/.cache/svlogd/others <&3'
```

A couple of things worth slowing down on:

- The leading `-` on the first `ExecStartPre` makes systemd tolerate `mkfifo` failing because the pipe already exists.
- The `exec 3<>%t/svlogd.pipe` opens the FIFO **read-write** and keeps that descriptor held open. This is the important trick: if `svlogd` only opened the pipe for reading, it would hit EOF and exit the moment the last writer disconnected. By holding a writer-end open ourselves, the pipe never sees EOF, so `svlogd` keeps draining it happily as individual services come and go.

With the collector running, every other service just points its output at the pipe:

```ini
StandardOutput=file:%t/svlogd.pipe
```

systemd opens the FIFO for writing, the service's lines flow into it, and the shared `svlogd` writes them to the rotating directory. One logger, one directory, no per-service bookkeeping.

This solves every previous problem — and, predictably, creates a new one. With everything funnelled into one stream, **you can't always tell which service a given line came from.** Plenty of programs don't print their own name, so the merged log is anonymous.

### 4.1. Attribution with `prefixer`

The fix is to tag each line at the source. [`prefixer`](https://github.com/navneetankur/prefixer) reads input, sticks a prefix on it, and writes it onward. Combined with `logpipe` so it doesn't linger:

```ini
ExecStart=logpipe prefixer onedrive - %t/svlogd.pipe -- /usr/bin/onedrive --monitor
```

Here `onedrive` is the prefix, `-` is stdin, `%t/svlogd.pipe` is where prefixed output goes, and everything after `--` is the actual program. Now the shared log shows you who said what, and `svlogd` still does the rotating.

## 5. The state we've reached

Tallying the books honestly:

- **All the chatter lives in one place.** When I actually need a service's output, it's in `~/.cache/svlogd/others`, rotated and bounded.
- **`journalctl -eb` is readable again.** It now mostly just records that those chatty services started and stopped — exactly the signal I wanted.
- The cost: the **unit files are busier** to read, since they carry `logpipe` and `prefixer` plumbing.
- And `svlogd` writes timestamps in **UTC**, so reading a log means doing timezone math in my head.

## 6. The UTC wrinkle

There are two reasonable ways out of the timestamp annoyance:

The clean one is to store TAI64N timestamps with `svlogd -t` and convert to local time only at read time. The utility for that — the one whose name I can never hold onto — is **`tai64nlocal`**. You pipe the log through it and it prints local time. The logs on disk stay timezone-neutral; only your eyes get the local conversion.

The blunt one is to make `svlogd` emit local time directly. There's a [patched build of runit](https://github.com/navneetankur/runit) where `-ttt` produces local time instead of UTC. It reads nicely with no conversion step — but before you reach for it, it's worth reading up on **why storing logs in local time is generally a bad idea** (DST jumps, ambiguous hours, and cross-machine comparisons all get unpleasant). Know what you're giving up before you give it up.

## 7. Bonus: ad-hoc processes

You don't have to write a unit file to get the same two benefits (RAM-usage view *and* captured logs) for a one-off process. `systemd-run --user` will wrap anything:

```sh
systemd-run --user logpipe prefixer chrome - $XDG_RUNTIME_DIR/svlogd.pipe -- google-chrome
```

That launches Chrome as a transient user unit, prefixed and piped into the same shared collector.

And if you only care about capturing the logs and not about the unit accounting, the shell already has you covered:

```sh
google-chrome &> $XDG_RUNTIME_DIR/svlogd.pipe
```

## 8. Was any of this necessary?

Mostly no. The honest answer to the original complaint was probably "section 1, `StandardOutput=file:` and move on." But the end state is genuinely pleasant to live in: a quiet journal, logs that exist when I want them, bounded disk use, and per-line attribution — at the price of some plumbing I only have to write once. If you've ever closed `journalctl` in mild irritation, that trade might be worth making. And if you haven't, well — now you have a non-existent problem to solve.
