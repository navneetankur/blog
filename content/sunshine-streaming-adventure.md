# Sunshine Streaming Adventure: Wayland to X11 and Back

*A journey through KMS capture, CRTC mysteries, and cherry-picked kernel commits*

---

I wanted to do something simple: stream a game from my desktop to my laptop. Moonlight and Sunshine exist exactly for this. Should be easy. It was not easy.

What followed was a multi-week detour through Wayland capture bugs, X11 screen tearing, modesetting driver source code, and one very confused ChatGPT session — all ending with the conclusion I least expected: Wayland works better for this than X11.

Here's the full story, in case you're stuck somewhere in the same maze.

---

## The Setup

- **Window manager:** Sway (Wayland) on the desktop, Moonlight on the laptop
- **GPUs:** Intel integrated + Nvidia discrete
- **Game launcher:** `uwu-run` / `prime-run` to force the game onto the Nvidia card
- **Capture method (target):** KMS

The first problem appeared immediately.

---

## Problem 1: The Disappearing Cursor

After starting the stream in Moonlight and switching to the game's workspace, the cursor would vanish. Not a showstopper, but annoying enough to investigate. I filed [an issue on the Sunshine repo](https://github.com/LizardByte/Sunshine/issues/5258). No progress came from that.

Digging deeper, I found the underlying cause: wlroots is mid-refactor on its capture methods, and Sunshine is simultaneously experimenting with a Vulkan wlroots renderer *and* a Vulkan encoder. Two moving targets. Breakage was inevitable and a fix wasn't coming soon.

**Decision: abandon Wayland for now, try X11.**

---

## Act II: X11 and the Screen Tearing Problem

Switching to i3 on X11 was straightforward. KMS capture worked, P1 (the cursor bug) was gone. Progress.

Then came screen tearing.

I vaguely remembered a Hacker News post about a `TearFree` option being added to the modesetting driver. I set the option. Nothing changed. More searching revealed the uncomfortable truth: the feature hadn't been released yet. It was sitting in the main branch of xf86-video-modesetting, not in any stable release.

So: what now?

### Cherry-Picking from Main

The options were: wait, switch to the main branch wholesale (risky), or surgically backport just the commits I needed. ChatGPT suggested cherry-picking. It sounded too clean to be true, but let's look.

I found the relevant developer's commits — nearly all of them, over a meaningful stretch of time, were tearfree work. Focused. Manageable. I cherry-picked them onto the release branch, resolved a few minor conflicts, and crucially verified that the **ABI version did not change**. That last part matters: if the ABI bumps, your input drivers (libinput, etc.) need to be rebuilt against the new version, and on a system with proprietary Nvidia drivers, that's a rabbit hole.

Built it. Ran it. Tearing gone. Simple, in the end.

*A few days passed.*

---

## Back to the Actual Goal: Streaming

Right. The game. Let me actually stream it.

**Intel display + `prime-run` game:** captured and streamed fine. P1 also resolved itself somewhere along the way — probably an update I'd applied before initially raising the issue.

**Nvidia display:** only the X11 capture method worked. KMS capture saw nothing.

Sunshine's own documentation discourages X11 capture — it's slow, CPU-heavy, a last resort. So this needed fixing.

---

## The CRTC = 0 Mystery

Sunshine's logs told the story: it couldn't even see the Nvidia connector. Disabling the Intel monitor with `xrandr` made it see *no* connectors at all.

Hypothesis: primary/secondary GPU confusion. I tried writing an Xorg config that referenced only Nvidia, no Intel. Restarted Xorg. Same result, except now Sunshine was capturing the TTY on the Intel monitor, and I'd lost the ability to use `xrandr` to disable it. An improvement in no direction.

`nvidia-settings` generated an Xorg config. Same result.

Sunshine could see `card0` and `card1` in the logs, but the connector it actually found — ID 125 — was the Intel one.

### Enter modetest

ChatGPT suggested running:

```bash
modetest -M nvidia-drm -c
```

Every connector showed `CRTC = 0`.

The implication: the Nvidia card isn't actively driving any display. KMS has nothing to capture because from KMS's perspective, the Nvidia card isn't doing anything. The proprietary X11 driver renders without going through KMS — it has its own path. So there's no KMS framebuffer to grab.

**Sway and Wayland, on the other hand, require KMS.** If Nvidia wants to support Wayland at all, it has to make KMS work. And if KMS is working, Sunshine can capture it.

---

## The Reluctant Return to Wayland

I hadn't successfully driven a monitor through the Nvidia card on Sway in a long time. Last time I tried, it was painful. But the logic was sound, so:

```
start sway → CRTC = 0 → monitor doesn't work
```

Expected. Then I poked at it. Config changes, kernel parameters, some combination of things. And then:

```
CRTC = 1 → monitor works on Sway
```

KMS capture: works.

Switched back to i3 to confirm: `CRTC = 0` again. Back to Sway: `CRTC = 1`.

The Nvidia card only properly exposes itself to KMS under Wayland. X11, with the proprietary driver, bypasses it.

---

## The Actual Fix: Kernel Parameter Placement

After enough digging, the root cause of the `CRTC = 0` issue came down to **where** the `modeset` parameter was being set.

The common advice is to put `options nvidia-drm modeset=1` in `/etc/modprobe.d/nvidia.conf`. Many wikis say Nvidia sets this by default now anyway.

Both of those statements are misleading in practice.

**What actually works:**

The `modeset=1` parameter needs to be set in the **bootloader**, not just in `modprobe.d`. The modprobe config doesn't reliably apply early enough for KMS to initialize correctly. Even if `cat /sys/module/nvidia_drm/parameters/modeset` says `Y`, that doesn't mean KMS initialized in time for the display to be active when Sunshine looks for connectors.

```
# In your bootloader kernel parameters:
nvidia-drm.modeset=1
```

Additionally: having Nvidia modules in initramfs may be needed in some configurations, though currently it works without them. Worth keeping in mind if things break after a kernel update.

---

## Summary: What Actually Matters

If you're trying to get Sunshine KMS capture working with an Nvidia card on Linux:

1. **Check CRTC status first.** Run `modetest -M nvidia-drm -c`. If all CRTCs are 0, KMS isn't working, and capture won't work. Fix that before debugging anything in Sunshine.

2. **`drm_info` is your friend.** It gives a readable view of what KMS knows about your display hardware.

3. **Set `nvidia-drm.modeset=1` in the bootloader**, not just in `modprobe.d`. The wiki saying it's the default is not the full story.

4. **Use Wayland.** Counterintuitive given Wayland's reputation for being less permissive about screen capture — but for Nvidia + Sunshine, Wayland forces KMS to actually work, which is what Sunshine's KMS capture path needs. X11 with the proprietary driver sidesteps KMS entirely.

5. **For Intel + `prime-run` games on an Intel-connected display**, X11 works fine for streaming. The Nvidia-connected display case is the tricky one.

---

## Postscript

The tearfree cherry-pick held up. Sway is stable. Sunshine captures cleanly via KMS. Moonlight on the laptop works.

Maybe one day I'll actually play the game.
