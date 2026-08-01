# Deadweight

**grab. lift. let go.**

A physics-based zombie survival game that runs in one HTML file. No engine, no
framework, no build step, no images, no audio files, no dependencies of any
kind — 3,160 lines of hand-written HTML, CSS, and JavaScript.

**▶ Play it: https://Herodotus08.github.io/deadweight**

Or download `deadweight.html` and double-click it. That's the whole install.

---

## The idea

You defend a shack. Zombies walk toward it. Your only real weapon is your
cursor: grab a zombie, lift it off the ground, and let go somewhere painful.

The catch is in the title. Damage comes from **impact**, not from clicking, so
every kill is a physics problem — how high did you get it, what did you drop it
on, and did you take anything else out on the way down. The bigger zombies are
literally dead weight: they fight your grip, they slip out, and the same grip
that holds a runner for six seconds holds a Warden for barely one.

## What's interesting under the hood

This started as a "can I do it with no assets" exercise and turned into the
project I'd actually want to talk through in an interview. The parts I'm proud
of:

### Verlet ragdolls, 7 points each

Every zombie is a physical body, not a sprite: 7 mass points (head, chest,
hips, two hands, two feet) joined by 7 distance constraints, integrated
positionally (`x, y, px, py`) and relaxed over `ITER = 6` passes per step at a
fixed `STEP = 1/60`. `GRAV = 0.62`, `DAMP = 0.994`. Limbs flail because they're
being solved, not because anyone keyframed them.

Bodies are only simulated when they need to be — `isSim(z)` gates the solver on
`grabbed || ragdoll || dead`, so a walking crowd costs almost nothing and the
frame budget goes to the bodies actually in the air.

### Three depth lanes on a 2D canvas

`LANES` gives each row a negative `dy` and its own scale, so the field has
depth and collisions only resolve *within* a lane. It's a cheap trick and it
completely changes how a crowded wave reads.

### The bug I'm most glad I found

`gripDrain()` decides how long a heavy zombie stays in your hand. It computes a
raw hold time from grip level, wave number, and a per-body `thrash` multiplier
rolled at spawn so no two bodies behave alike.

The original clamped the floor *after* multiplying by `thrash`:

```js
return Math.max(0.12, (base + late - over * 0.06) * z.thrash);   // wrong
```

At max grip the raw figure went negative, the floor swallowed it, and `thrash`
stopped mattering entirely — **every heavy in the wave held for exactly 8.3
seconds.** The one system built to stop players counting frames had become the
most countable thing in the game.

The fix is a one-line reorder — clamp first, then apply `thrash`:

```js
return Math.max(d.floor, d.base + late - over * 0.06) * z.thrash;  // right
```

It's invisible on a code read. I only caught it by printing a table of hold
times across grip levels and noticing a column of identical numbers.

### Time-windowed slam damage

Bosses were dying in two hits because a single landing spans several frames and
each frame was independently applying capped impact damage — the cap leaked
once per frame. `SLAM_WINDOW = 0.42` plus a per-body `slamBudget` groups a whole
landing into **one** capped event. A full-strength throw now takes 7.9% of a
Warden's health against a 15% cap, and a Warden takes about nine good throws.

### Armour that ignores your hands, on purpose

`hurt(amount, kind)` routes damage by source. `t.armor` reduces `'gadget'` and
`'blast'` only; `'impact'` bypasses it completely. So gear gets worse against
armoured zombies while your hands stay honest, and the slam cap — not armour —
is the only thing limiting throw damage. Helmets are the single exception
(`amount *= 0.45` on impact).

### Every sound is synthesised

No audio files. Groans, bone snaps, explosions, and the menu music are all
built at runtime from Web Audio oscillators and noise buffers.

### A balance pass with an actual autoplayer

Rather than guess at difficulty I wrote a headless harness (Node's `vm` plus a
fake DOM and a stub canvas) and a bot with a tunable `skill` parameter, then
measured where runs actually end:

| bot skill | outcome over repeated runs |
|-----------|----------------------------|
| 1.0 (acts every frame) | clears wave 25 untouched |
| 0.5 | 5 of 6 runs cleared 20+, one death at 22 |
| 0.4 | dies waves 21–24 |
| 0.3 | dies around wave 17 |
| 0.2 | dies waves 14–18 |

That's the band I wanted: superhuman play is unbounded, good-but-imperfect play
dies in the low twenties.

The same harness rendered all 9 zombie types × elite × 3 lanes × 2 facings × 4
animation states — 432 draw passes — asserting zero `NaN` or `Infinity` ever
reaches a canvas call.

### Economy with real ceilings

Every upgrade caps: Grip 10, Bone Breaker 5, Shack 10, and gadgets capped by
**count** (6 grenades, 4 spikes, 4 saws) with level caps set so the last
purchasable level is the one that actually grants the item — no trap purchases.
The combo multiplier pays out to a maximum of 2.8×, because uncapped it chained
30+ kills in a late wave and left runs sitting on six figures of gold with
nothing to buy. A scaling Repair Shack gives that late-game gold somewhere to go.

## Controls

| | |
|---|---|
| **Drag** | Grab a zombie, lift, fling |
| **1 / 2 / 3** | Grenade · Spike trap · Saw blade |
| **ESC** | Pause / menu |

Heavies and bosses need a minimum grip level before you can even attempt a
lift, and each attempt can slip. Failed attempts build `strain`, which makes
the next attempt likelier — so wrestling is a strategy, not a coin flip.

## Running it

Open `deadweight.html` in any modern browser. There is no server, no `npm
install`, no toolchain. Chrome, Firefox, Edge, and Safari all work.

## Credits & legal

Written by **Dale D.**, 2026.

Inspired by *ZombieSmash!* (GameDoctors / Zynga, 2012). This is an original
implementation and contains **no code, art, audio, or assets** from that or any
other game — every visual is drawn procedurally with Canvas 2D and every sound
is generated at runtime with Web Audio. Not affiliated with, endorsed by, or
sponsored by Zynga Inc. or GameDoctors. All third-party trademarks belong to
their respective owners.

© 2026 Dale D. All rights reserved — see [LICENSE](LICENSE). You're welcome to
read the source and play the game; please don't redistribute it or pass it off
as your own.
