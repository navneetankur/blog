+++
title = "When SSH Steals Your Seat: systemd User Manager, Logind Sessions, and Sway"
date = 2026-06-22
+++

There is a subtle trap waiting for anyone who remotely wakes their desktop, SSHs in to check
on it, and then walks over to use it physically. Everything looks fine — you are logged in, the
TTY is there, your user services are running — but `systemctl --user start sway` fails with a
cryptic complaint about not finding a seat. The culprit is the systemd user service manager
quietly belonging to your SSH session instead of your TTY session, and the fix requires
understanding how logind tracks session ownership.

---

## The Workflow That Triggers This

The setup is ordinary enough:

- Desktop is off. Send a Wake-on-LAN magic packet from the laptop.
- SSH into the desktop as soon as `sshd` is online, to do some work before physically walking over.
- Don't close that SSH terminal.
- Walk to the desktop, turn on the monitor.
- From the physical keyboard on TTY1, try to start Sway as a user service.
- It fails.

The failure is not random. It is a deterministic consequence of the order in which things
started.

---

## How the systemd User Manager Gets Attached to a Session

systemd runs one user manager instance per logged-in user: `user@UID.service`. This instance
is shared across all sessions for that user. Logind controls when it starts: it is activated
on first login and kept alive until the last session ends.

The important detail is *which* session logind associates the user manager with. Logind uses
cgroups to track this. When the manager starts, it is placed under the scope of the session
that triggered it. Later sessions for the same user — including a TTY login — reuse the
already-running manager but do not reassign its session affiliation.

So: whichever session logs in first *owns* the user manager. Everything running under it
inherits that session's identity.

---

## The Race With Autologin

Most people who sit at a Sway desktop use autologin on TTY1. Getty starts, PAM logs the user
in automatically, and a TTY session is created. Normally this would be the first session, and
the user manager would belong to it.

But Wake-on-LAN and SSH create a race.

When the machine boots:

1. `sshd` comes online, often before the display manager or getty has finished autologin.
2. You SSH in. PAM creates an SSH session. The user manager does not exist yet, so systemd
   starts it now — under the SSH session.
3. Some time later, autologin completes on TTY1. A TTY session is created. But the user
   manager is already running, already owned by SSH. Logind does not reassign it.

You have to SSH quickly enough — before autologin finishes — for this race to resolve in
SSH's favour. With autologin enabled and a fast boot, this is not difficult to hit. The
window is small but reproducible.

Once you have walked to the desktop and are working on TTY1, the user manager is still
quietly associated with the SSH session. Nothing on screen tells you this.

---

## Verifying the Problem

From the physical TTY1, run these two commands:

```
systemd-run --user --scope loginctl session-status
```

```
loginctl session-status
```

The first spawns a transient scope under the user manager and asks logind which session it
belongs to. It will report the SSH session.

The second asks logind directly which session the current shell process belongs to. It will
report the TTY session.

They are different. Your shell is in the TTY session, but the user manager lives in the SSH
session. Any user service you start inherits the SSH session's environment and seat
assignment — which is to say, no seat at all.

---

## Why Sway Specifically Fails

Sway uses wlroots as its backend. wlroots needs to open DRM and input devices. It does this
through a *seat*. On a logind system, the seat is provided by logind itself; on a seatd
system, by the `seatd` daemon.

When Sway starts as a user service, it inherits the user manager's environment. That
environment came from the SSH session. SSH sessions have no seat — there is no physical
display device attached to a network connection. So `XDG_SEAT`, `XDG_VTNR`, and
`XDG_SESSION_TYPE` are either absent or wrong.

wlroots tries to find a seat, finds nothing it can use, and exits. The error message mentions
seat or tty, depending on the wlroots version, but the root cause is always the same: the
session context says there is no physical seat.

---

## Solutions

There are several ways out, ranging from crude to elegant.

### 1. Close All SSH Sessions

When the last SSH session closes, logind reassigns the user manager to the next remaining
session. If the only other session is TTY1, the manager's session affiliation updates to the
TTY session, and `XDG_SEAT` and friends become correct.

Workable as a one-time fix but annoying if you want a persistent SSH connection open.

### 2. Kill the SSH Session From the Desktop Side

Same effect as above, without touching the laptop:

```bash
loginctl terminate-session <SSH_SESSION_ID>
```

Get the SSH session ID from `loginctl list-sessions`. This is more surgical and works while
keeping other things running.

### 3. Start Sway Directly From the TTY

Skip the user service entirely:

```bash
exec sway
```

When you run this from your TTY1 shell, the process is a child of your TTY session. It
inherits the correct environment — `XDG_SEAT`, `XDG_VTNR`, `XDG_SESSION_TYPE=tty` — because
those were set by PAM when autologin created the TTY session.

wlroots finds the seat, opens the DRM device, and everything works. The tradeoff is losing
service-level supervision: no `Restart=`, no dependency ordering against other user services.

### 4. Use `systemd-run --user --scope`

Running Sway inside a transient scope works even with the SSH-owned manager:

```bash
systemd-run --user --scope \
  --setenv=XDG_SEAT="$XDG_SEAT" \
  --setenv=XDG_VTNR="$XDG_VTNR" \
  --setenv=XDG_SESSION_TYPE="$XDG_SESSION_TYPE" \
  sway
```

When called from your TTY1 shell, your current environment has the correct seat variables.
Explicitly passing them into the scope overrides whatever the user manager's environment
contains. wlroots sees a valid seat and starts normally.

This works because Sway can take the seat from environment variables when it cannot get them
from the session. The `--scope` execution context is enough to satisfy it.

### 5. Start seatd

If `seatd.service` is running, wlroots uses the seatd protocol instead of going through
logind for seat access. seatd does not care about session attribution — it manages seat
device access as a standalone daemon.

```bash
sudo systemctl enable --now seatd.service
# make sure your user is in the 'seat' group
sudo usermod -aG seat "$USER"
```

Then `systemctl --user start sway` works regardless of which session owns the user manager,
because wlroots is no longer asking logind for seat permissions at all.

This is a clean long-term fix if you want Sway as a user service and are comfortable
replacing logind seat management with seatd.

### 6. Import the Correct Environment Into the User Manager

The most precise fix, without changing how seat management works:

```bash
systemctl --user import-environment \
  XDG_SESSION_ID \
  XDG_SESSION_TYPE \
  XDG_SESSION_CLASS \
  XDG_SEAT \
  XDG_VTNR
```

Run this from your TTY1 shell before starting Sway. It copies those variables from your
current shell — which has the correct TTY values — into the user manager's global environment
block. Any unit started after this point, including Sway, inherits the corrected values.

`XDG_SEAT` alone is often sufficient for wlroots to find the seat. If Sway still complains,
import the full set above.

To automate this on every TTY login, add to `~/.bash_profile`:

```bash
if [[ "$XDG_SESSION_TYPE" == "tty" && -n "$XDG_SEAT" ]]; then
    systemctl --user import-environment \
        XDG_SESSION_ID \
        XDG_SESSION_TYPE \
        XDG_SESSION_CLASS \
        XDG_SEAT \
        XDG_VTNR
fi
```

This only runs when the shell is in a real TTY session with a seat, so it is safe to leave in
permanently without affecting SSH logins.

---

## Summary

| Solution | Closes SSH? | Needs root? | Persistent fix? |
|---|---|---|---|
| Close all SSH sessions | Yes | No | No |
| Kill SSH session via loginctl | No | No | No |
| Direct `exec sway` from TTY | No | No | No (no service) |
| `systemd-run --user --scope` | No | No | No (manual each time) |
| Enable seatd | No | Yes | Yes |
| `import-environment` in profile | No | No | Yes |

The import-environment approach in `.bash_profile` is the least invasive permanent fix if you
want to keep Sway as a user service and keep SSH sessions open. The seatd approach is equally
good if you prefer seatd over logind for seat management. Everything else is situational.

The underlying lesson: the systemd user manager is a per-user singleton, not a per-session
one. Whichever session starts it first keeps it. On a machine with autologin and a habit of
SSH-before-sitting-down, that will almost always be the SSH session.
