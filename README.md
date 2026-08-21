# RedM: `fwDynamicArchetypeComponent` pool increase — test results

Test requested by the Cfx team on Discord (18/08/2026). Run 21/08/2026.
Server: Atlanta Roleplay (RedM). Client build b1491. GPU: AMD RX 6700 XT.

## TL;DR

**Increasing `fwDynamicArchetypeComponent` works in RedM, and the increase is
honored in full. No new crash types were observed.**

- Default ceiling reproduces the known `RAGE error: 0x9952DB5E:212` at
  `gta-core-rdr3.dll+38C98`, fired on `Performing deferred RELOAD_MAP_STORE`.
- With `increase_pool_size "fwDynamicArchetypeComponent" 1000`, the measured
  ceiling moved by **975–1125 archetypes** for a requested 1000.
- Measured twice, on two different resource baselines, with consistent results.
- The only crash observed at any point was the same 212, and only when the new
  (higher) ceiling was exceeded. No memory, streaming or other pool crashes.

This is the failure that took our production server down on 26/07/2026: every
connecting client crashed at join. We are the largest RedM server; this pool is
our hard ceiling on custom map/prop content.

## Method

We measure the ceiling by stacking **phantom archetypes**: N `CBaseArchetypeDef`
entries in one ytyp, all pointing at the **same** 1 KB `.ydr`.

Rationale: an earlier bisect (26/07) established that sharing `assetName` across
archetypes does **not** reduce pool usage — the counter is ityp *entries*, not
`.ydr` files. That makes shared-asset archetypes useless as an optimization and
ideal as filler: N phantoms cost N slots and ~0 disk.

Because the ceiling is `base + ghosts_at_crash`, and the resource baseline is
identical on both sides of the comparison, **the baseline cancels out in the
delta**. Production parity is therefore not required to measure the increase.

Each probe: regenerate the ytyp → `refresh` → `restart` the resource → observe.

Verification that a probe actually took effect (we do not trust "it didn't
crash"): `IsModelValid(joaat('atl_pool_%05d'))` on the client for N and for
**N+1**. N must be true and N+1 false — without the N+1 control the probe does
not discriminate.

## Results

Baseline A:

| config | ghosts | result |
|---|---|---|
| no increase | 150 | ok |
| no increase | 225 | ok |
| no increase | 300 | **CRASH 212** |
| `+1000` | 900 | ok |
| `+1000` | 1200 | ok |
| `+1000` | 1275 | ok |
| `+1000` | 1350 | **CRASH 212** |
| `+1000` | 1500 | **CRASH 212** |

⇒ ceiling without increase = `base_A + [225, 300)`
⇒ ceiling with `+1000`     = `base_A + [1275, 1350)`
⇒ **delta ∈ (975, 1125) for a requested 1000**

Baseline B (larger, more map resources loaded):

| config | ghosts | result |
|---|---|---|
| `+1000` | 600 | ok |
| `+1000` | 900 | ok |
| `+1000` | 1100 | ok |
| `+1000` | 1200 | ok |
| `+1000` | 1300 | **CRASH 212** |

⇒ ceiling with `+1000` = `base_B + [1200, 1300)`, consistent with A.

## Evidence

Client log, increase active:

```
Pool size increase validation failed: it is not allowed to change size of pool
fwDynamicArchetypeComponent. However the "moo 31337" is set, so the validation
error will be ignored.
...
Performing deferred RELOAD_MAP_STORE.
Error: RAGE error: 0x9952DB5E:212
```

`crashometry.json`, increase active:

```
crash_hash                                    gta-core-rdr3.dll+38C98
crashometry_invalid_pool_size_increase_used   fwDynamicArchetypeComponent: 1000
```

`crashometry.json`, control run (increase disabled): the
`invalid_pool_size_increase_used` key is **absent**, confirming the control was
genuinely running without the increase.

Resource restart exercises the same path as a fresh join, and unloads cleanly:

```
removing DLC_ITYP_REQUEST .../atl_pooltest.ytyp
done removing ...
loading DLC_ITYP_REQUEST .../atl_pooltest.ytyp
loading CFX_PSEUDO_ENTRY RELOAD_MAP_STORE
Performing deferred RELOAD_MAP_STORE
```

## Three findings about the test procedure itself

These cost us several wasted cycles and will likely affect anyone else you ask
to run this.

**1. `--` is not a comment in FXServer cfg files.** The Discord instructions use
Lua-style comments:

```
set moo 31337 -- turn on dev mode
```

Pasted literally, `set` receives seven arguments instead of two and the convar is
never created (`GetConvar('moo')` returns empty). Cfg comments are `#`. We spent
two cycles measuring the default ceiling while believing the increase was active.

**2. The client rewriting `CitizenFX.ini` is a symptom, not the bug.** While (1)
was in effect, the client stripped `fwDynamicArchetypeComponent` from
`PoolSizesIncrease` on every join, and the "RedM needs to restart" dialog listed
the other six pools without ours. This looks like the client forgetting the
setting; it is actually the server not announcing the pool. Once the server
announces it correctly, **the client writes the entry itself** — exactly as it
does for the other six pools, none of which were ever edited by hand.

Suggestion: the manual ini step may be masking how many testers are failing at
the server step without realising it.

**3. A client restart loses `+set moo 31337`, and the pool dialog triggers one.**
Measured on both sides:

```
server: GetConvar('moo') = 31337
client: GetConvar('moo') = 0      (after the restart the dialog triggered)
```

The loop: the pool set changes → client shows the restart dialog → client
restarts → the restart does not preserve the command-line `+set moo 31337` →
without `moo`, `Sanitize` rejects the non-allowlisted pool → it is dropped from
the negotiated set and wiped from the ini.

Working procedure: close the client fully, edit the ini to match the server
exactly, launch via the shortcut carrying `+set moo 31337`, connect. **The valid
run is the one where no dialog appears at all** — if the dialog shows, `moo` is
already lost.

## Caveats

- Our ruler is a disk census of ityp entries and has a known offset relative to
  what the engine actually loads, so absolute totals carry roughly ±10–20. The
  **delta** does not depend on this.
- Terrain LOD ytyps (`*lod_combine_metadata*`, `nopq_*`, `rstu_*`) do **not**
  consume this pool — verified separately by loading 854 such archetypes with no
  effect on the ceiling. Any census must exclude them.
- We tested `+1000`, not the 30000 used on FiveM. We did not observe clamping at
  this magnitude, but we cannot extrapolate to 30000 without measuring it.
- Test ran on a local server whose resource set is close to, but not identical
  with, production (escrow-protected map resources do not start under a
  different license key, ~86 archetypes short).

## Ask

Please consider adding `fwDynamicArchetypeComponent` to the RedM pool-size
allowlist at `gss.cfx-services.net/v1/public/pool-size-limits/redm`. FiveM
already carries it at 30000; RedM has no entry.

We are happy to run further tests — including a `+20000` run — if useful.
