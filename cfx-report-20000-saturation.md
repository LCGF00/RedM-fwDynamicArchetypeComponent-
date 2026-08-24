# RedM: `fwDynamicArchetypeComponent` at `+20000` — silent loss of vanilla content

Follow-up to the `+1000` test (`cfx-report-fwdynamicarchetypecomponent.md`).
Same server, same client build (b1491), same ghost-archetype method. Run 21/08/2026.

## TL;DR

**`+20000` does not crash — and that is the problem.** Our archetypes keep
registering all the way to 20000, but somewhere past ~11250 the game starts
**silently dropping its own content**: first water, then terrain, then other
custom resources. No crash, no log line, nothing in crashometry. Physics stays
intact, so the player walks on invisible ground. This is strictly worse for an
operator than the crash at `+1000`: you ship content that "fits" and the map
breaks for every player with no signal anywhere.

**Stable range with `+20000`: up to ~11250 ghost archetypes on top of our
baseline; degraded from 12500.** With our baseline estimated at 800–900, that is
a usable total of roughly **12 000 archetypes — about 11× stock, not 21×.**

## Setup

- `server.cfg`: `set moo 31337` + `increase_pool_size "fwDynamicArchetypeComponent" 20000`
- `CitizenFX.ini`: `"fwDynamicArchetypeComponent":20000`
- Client launched via shortcut with `+set moo 31337`; a valid run shows **no**
  pool dialog (see the `+1000` report for why).
- Every result below comes from a **full server restart + client relaunch + clean
  join**. See "Method corrections" for why.

## Results

| ghosts | crash | water | vanilla terrain | custom resource (`atlanta_estruturas`) | vanilla props |
|---|---|---|---|---|---|
| 3000 | no | — | — | — | — |
| 10000 | no | ✓ | ✓ | ✓ | ✓ |
| 11250 | no | ✓ | ✓ | ✓ | ✓ |
| **12500** | no | **✗** | ✓ | ✓ | ✓ |
| 15000 | no | ✗ | **✗** | ✓ | ✓ |
| 19000 | no | ✗ | ✗ | **✗** | ✓ |
| 20000 | no | ✗ | ✗ | — | ✓ |

("—" = not measured on that run; the canary was introduced after 3000.)

Ghost archetypes registered in full on **every** row — `IsModelValid` true for
index N and false for N+1, including at 20000.

**Order of sacrifice: water → terrain → other custom resources.** Props from the
base game's always-loaded ityps (`p_chest02x`, `p_barrel02x`) survived on every
run. Content that streams per region — terrain tiles, water — went first.

**Physics is intact.** The player can walk across the invisible terrain; only the
drawable is gone. Collision lives in static bounds (a separate, allowlisted pool),
which is what makes this easy to miss on a live server.

### What it looks like

- 12500: the lake around the player drains; fish hang suspended over the exposed
  lakebed (vanilla terrain, present). Custom island and buildings render normally.
- 15000: vanilla ground disappears; water to the horizon where terrain should be.
  Custom decking the player stands on is present.
- 19000: a custom masonry building is missing from a custom map; `atl_wall_alv_2m`
  and `atl_canto` (our construction kit) report invalid.

## Method corrections

Two things that would have produced wrong conclusions had we not caught them.

**1. Restarting a resource with a player online under-registers archetypes.**
Same files, same server: 10 000 ghosts registered as **9413** via
`restart` + `RELOAD_MAP_STORE`, and as **10 000** on a clean join. 15 000 ghosts
via restart: 9413 again. We initially read this as the increase saturating at
~9400. It was an artifact of the live-reload path. **All results above are from
clean joins only.** Anyone measuring this pool by restarting a resource will
report a lower ceiling than the real one.

**2. "Our archetypes loaded" proves nothing.** At 15 000 every ghost registered
while the ground under the player's feet was gone. The check that matters is
whether **vanilla** content survived. We used the terrain tile under the player
as a canary — `IsModelValid(joaat('p_09__hd_0_2_-3'))`, tile name looked up from
the game's own ymap index by player position. It discriminates both ways: true at
10 000 and 11 250, false at 15 000, same tiles, same spot. Water was assessed
visually (screenshots); the `GET_WATER_HEIGHT` native returned false at that spot
regardless of whether water was visibly present, so it was excluded as a probe.

## Interpretation

With `+1000`, exceeding the ceiling **crashed** (the pool's own overflow path).
With `+20000`, nothing crashed even at 20 000 ghosts — the pool itself is large
enough — but a **second, shared limit** starts to bite around ~11 250 extra
archetypes, and it bites the base game first. We do not know which structure that
is. The obvious candidate is whatever global archetype map vanilla and custom
archetypes share (PoolManagement names `fwArchetypePooledMap` next to this pool),
but we have not measured it and are not asserting it.

The practical consequence for the allowlist: **approving
`fwDynamicArchetypeComponent` alone at 30 000 (the FiveM value) would let
operators walk straight into silent vanilla-content loss.** Either the second
limit needs to move with it, the allowlisted value should sit near the usable
ceiling (≈ +10 000 on our measurements), or at the very least the drop should be
reported in the log instead of being invisible.

## Caveats

- Local server; resource set close to production but ~86 archetypes short
  (escrow-protected map resources do not start under a different license key).
- Our baseline is estimated (800–900), not measured to the unit. The ghost counts
  are exact; the absolute totals carry that uncertainty.
- Water was judged by eye and screenshot, not by a native — see above.
- We stopped bisecting between 11 250 (good) and 12 500 (degraded). That is
  enough precision for the conclusion; we can tighten it if useful.

## Repro

1. Generate N ghost archetypes (all pointing at one 1 KB `.ydr`) across a few ytyps.
2. `set moo 31337` + `increase_pool_size "fwDynamicArchetypeComponent" 20000` on
   the server; matching ini entry and `+set moo 31337` on the client.
3. Full server restart, client relaunch via the shortcut, join with no dialog.
4. Check: (a) `IsModelValid` on ghost N and N+1; (b) `IsModelValid` on the
   terrain tile under the player; (c) look at the nearest body of water.

Generator and split script available on request.
