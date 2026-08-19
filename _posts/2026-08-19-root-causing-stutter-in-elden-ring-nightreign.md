---
title: "Root-Causing Stutter in Elden Ring Nightreign"
description: "Root-causing an Elden Ring Nightreign stutter on an Intel hybrid CPU."
---

I've got 225 hours in Elden Ring Nightreign and a solid 25 hours is probably just debugging and testing mods.
Recently, I've gone mobile and taken the game on my laptop. It's been great for the most part, but had intermittent stutters
on my Intel Core Ultra 9 285H, regardless of what settings I used. They happened often enough to make the game
harder and harder to play on top the difficulty of the game itself. I was running the same mod pack as my friend and
yet only I had these problems. I wasn't sure what the root cause was but I suspected the mods exacerbated it.

But the stutter turned out to be unrelated to the mods, even when playing vanilla I had it. I could reproduce it easily in the lobby, but
fixing it would mostly be me spamming Codex.

## What didn’t pan out

I tried lowering the FPS limit, going back to the stock 60 FPS cap, changing
NVIDIA latency settings, switching between fullscreen and borderless,
removing my FPS/FOV mod, and running a much smaller mod profile with
only Seamless Co-op.

Codex and I had plenty of possible explanations, but no way to tell them apart.

## Turning a workaround into a repro

I ended up going old fashioned and found [a Reddit post about an Elden Ring stuttering
fix](https://www.reddit.com/r/Eldenring/comments/1aqv99t/potential_stuttering_fix_for_certain_players_cpu0/)
that suggested removing CPU 0 from the game’s affinity in Task Manager. I tried
it in Nightreign and the stuttering disappeared.

I thought that was that so I installed Process Lasso, set the same CPU 0 rule,
restarted the game, and the stutter came back. I toggled CPU 0 on while
the game was running, and it became smooth again.

One comment described the same thing: CPU 0 could be toggled without
bringing the stutter back. Leaving CPU 0 disabled wasn't as important as just changing the affinity at all.

That gave me a cheap repro: I could reliabily go from bad to good with one quick action in the same session. I
didn’t know what the toggle changed, but now we could collect evidence without
guessing.

## From a repro to a trace

I took that repro back to Codex and it suggested recording a Windows performance
trace across the bad state and the affinity toggle, then comparing the two.

I didn’t know what an ETW trace was or what I was supposed to find in one but I ran the commands,
recorded it, marked the toggle, and handed it back to Codex for analysis.

One worker stood out to Codex: `EWP_LOW_Single_Clus1`. Before the toggle, it only ran on
logical CPUs 1–3 and repeatedly sat ready but waiting to be scheduled. After
the toggle, those delays disappeared.

| Measurement | Before the toggle | After the toggle |
|---|---:|---:|
| Logical CPUs observed | 1–3 only | 0–13 during the sample |
| Ready delays at least 5 ms | 26 | 0 |
| Ready delays at least 16.667 ms | 15 | 0 |
| Maximum ready delay | 60.887 ms | 1.751 ms |

That still didn’t tell us who limited the worker. It could’ve been Windows, a
driver, a mod, or the game itself.

So we made a diagnostic DLL that logged the relevant
Windows API calls inside `nightreign.exe` without changing them. I was inspired by wrapping externally resolved calls through `LD_PRELOAD` and the log showed Nightreign starting with the full process mask, `0xFFFF`. About
2.7 seconds into startup, Nightreign called `SetThreadAffinityMask` with
`0xE`, limiting a thread to logical CPUs 1, 2, and 3. Two milliseconds later,
that thread named itself `EWP_LOW_Single_Clus1`.

Nightreign restricted this worker to a particularly bad CPU set: CPU 1 was fast and unparked, while CPUs 2 and 3 were
slower and parked. When the worker got busy, it waited tens of milliseconds
for CPU time.

The Task Manager toggle worked because it happened after Nightreign restricted
the worker. Process Lasso applied its rule at launch, so Nightreign’s later
per-thread change won.

## The fix

The fix is a small native DLL that waits for `EWP_LOW_Single_Clus1`, reads
Nightreign’s process affinity, and restores only that thread to the full
process mask.

In testing, the DLL logged just the one correction: the worker moved from
`0xE` to the full `0xFFFF` process mask. The stutter was gone!

The source, DLL, and installation instructions are in the
[Nightreign Intel affinity patch repository](https://github.com/oskarwirga/nightreign-intel-affinity-patch).
I tested it with Nightreign 1.3.3.0 on a Core Ultra 9 285H. It targets this
specific affinity problem. I only use it with an offline, modded launch and not through the official anti-cheat path.

I still don’t know how to read an ETW trace, but all I need is good repro!
Once I had that, I could let Codex collect the evidence, analyze the result, and
narrow the problem until we had a fix we could explain and verify.
