# Minecraft Super Factory Manager (SFM) Guide

> Making and modifying SFML programs (`*.sfm` / `*.sfml`). Assembled 2026-09-15 from [Relvl/mc-sfm-codebook](https://github.com/Relvl/mc-sfm-codebook), [sfmhub.site/guide](https://sfmhub.site/guide), official/community sources, and the three `.sfml` programs in this repo. Claims marked `[INFERENCE]` or `version-dependent` need confirmation against the [SFML grammar](https://github.com/TeamDman/SuperFactoryManager/blob/1.19.2/platform/minecraft/src/main/antlr/sfml/SFML.g4).

## Part A — First Program, Execution Model

Scope: one program moving one stack. Full syntax is Part B; editing an existing repo program is Part C.

A program is a `NAME` plus one or more trigger blocks; labels referenced by `FROM`/`TO` are declared in code.

Keywords are case-insensitive. `--` starts a line comment. Every operation ends with `forget`.

### First program

Moves one cobblestone from `chest-a` to `chest-b` about once per second:

```sfml
NAME "hello-world"

-- EVERY <n> TICKS: timer trigger. 20 ticks ~= 1 s at 20 TPS.
EVERY 20 TICKS DO
  INPUT 1 cobblestone FROM chest-a
  OUTPUT 1 cobblestone TO chest-b
  FORGET
END
```

### Execution model

- Program = `NAME` plus one or more trigger blocks (`EVERY <n> TICKS`, `EVERY TICK`, redstone pulse [INFERENCE: exact pulse keyword varies by version — confirm against `SFML.g4`]).
- Triggers run in written order, top to bottom, each time they fire.
- Each trigger block = ordered statement list: `INPUT` / `OUTPUT` / `IF` run top to bottom.
- Per-trigger input list: `INPUT` statements register what the trigger pulls this pass; the list is cleared after the trigger finishes, so each pass re-declares its inputs.
- `FORGET` terminates each operation; omitting it leaves the previous operation open and the program fails to load.
- Timing assumes 20 TPS: `20 TICKS` ~= 1 s, `1200 TICKS` ~= 1 min. Minimum loop interval is 20 ticks [INFERENCE: codebook reports a 20-tick minimum; shorter intervals may be clamped or rejected per version].
- One trigger pass is: gather inputs in order, route to outputs in order, clear inputs, wait for next trigger.

---

## Part B — SFML Syntax Reference + Pattern Catalogue

Conventions for this part: keywords are case-insensitive (`input` = `INPUT`).
Line comment marker is `--`. Every I/O operation inside a trigger block is
terminated by a bare `forget` line. All snippets below are grounded in the
local repo files (`Mekanism4xOreProcessing.sfml`, `ModularBeesHive.sfml`,
`ModularBeesCentrifuge.sfml`) and the codebook/SFMHub findings; forms not
present locally are marked with their source.

### 1. Program skeleton and NAME header

Every program starts with a `NAME` header, then one or more trigger blocks
each closed by `END`:

```sfml
NAME "Mekanism 4x Ore Processing"
EVERY redstone pulse DO
    INPUT *Raw* FROM Interface
    OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
END
```

```sfml
NAME "Modular Bees Hive"
--Modified from BucketSt's Modular Hive
EVERY 20 TICKS DO
    INPUT FROM Hive
    OUTPUT *Honey* TO Storage
forget
END
```

Rules:

- `NAME "<string>"` appears once, first line.
- `--` starts a line comment (used throughout all three local files).
- `forget` terminates each INPUT/OUTPUT operation, not each block. Omitting
  it merges operations. (Codebook: forget-after-every-op.)

### 2. Triggers

Observed trigger heads (all closed by `END`):

```sfml
EVERY redstone pulse DO
END
```

```sfml
EVERY 20 TICKS DO
END
```

```sfml
EVERY TICK DO
    INPUT fe:: FROM Power TOP SIDE
    OUTPUT fe:: TO Overclocker
END
```

- `EVERY redstone pulse DO` — used by both Mekanism and Centrifuge programs
  for all item/fluid/gas moves.
- `EVERY 20 TICKS DO` — Hive bulk program (codebook: 20-tick minimum loop).
- `EVERY TICK DO` — energy-only block in all three local files.
- `EVERY n TICKS` / `EVERY n SECONDS` counted forms are per codebook/SFMHub;
  `n` is an integer (`EVERY 20 TICKS`, `EVERY 5 SECONDS`). [INFERENCE: exact
  upper bounds and sub-tick scheduling are version-dependent.]

### 3. INPUT reference

General shape:

```sfml
INPUT [filter] FROM [EACH] <label>[, <label>...] [<SIDE> SIDE] [SLOTS n-m]
```

#### 3.1 Omitted filter (move-all)

No filter = match everything the source exposes on that resource type:

```sfml
INPUT FROM Hive
OUTPUT *Honey* TO Storage
forget
```

```sfml
INPUT FROM HoneyCentrifuge SLOTS 0-8
OUTPUT TO Storage
forget
```

Second form scopes the match to slots (Centrifuge output slots). Pair with a
bare `OUTPUT TO` to drain without filtering.

#### 3.2 Single ID and explicit `item::` namespace

```sfml
INPUT gem FROM Hive
OUTPUT gem TO Trash
forget
```

Explicit item namespace (codebook-attested; no local occurrence):

```sfml
INPUT item::minecraft:iron_ingot FROM Chest
OUTPUT item::minecraft:iron_ingot TO Furnace
forget
```

`item::` forces item-type matching; bare `gem` above relies on the default
item type (see §7).

#### 3.3 Lists (comma-separated filters)

```sfml
OUTPUT EXCEPT *comb*,gem TO Storage
```

Filters (and `EXCEPT` lists) are comma-separated. Input-side equivalent:

```sfml
INPUT iron_ingot,gold_ingot FROM Chest
OUTPUT iron_ingot,gold_ingot TO Furnace
forget
```

#### 3.4 Wildcards and mod wildcards

Substring/prefix/suffix globs observed locally:

```sfml
INPUT *Raw* FROM Interface
OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
```

```sfml
INPUT electrode* FROM Resupply
OUTPUT electrode* TO Overclocker
forget
```

```sfml
INPUT *Jelly FROM Resupply
OUTPUT *Jelly TO TreaterB
forget
```

Mod-scoped wildcard (codebook-attested):

```sfml
INPUT item:minecraft:* FROM Chest
OUTPUT item:minecraft:* TO Storage
forget
```

`*` matches within the resource-type namespace on that side.

#### 3.5 EXCEPT deny-list

`EXCEPT` follows INPUT/OUTPUT directly; list is comma-separated:

```sfml
INPUT FROM Hive
OUTPUT *Honey* TO Storage
OUTPUT *comb* TO BufferStorage
OUTPUT EXCEPT *comb*,gem TO Storage
forget
```

Semantics: after the two positive outputs drain honey and combs, the
`EXCEPT` line drains everything else except combs and gems.

#### 3.6 Typed non-item filters: `fluid::` / `gas::` / `fe::`

Bare typed filter = all of that type; suffixed form filters within it:

```sfml
INPUT fluid:: FROM Hive
OUTPUT fluid:: TO Storage
forget
```

```sfml
INPUT fluid:: FROM HoneyCentrifuge
OUTPUT fluid:: TO Storage
forget
```

```sfml
INPUT gas::*oxygen* FROM Interface
OUTPUT gas::*oxygen* TO Purification TOP SIDE
forget
```

```sfml
INPUT gas::*chloride* FROM Interface
OUTPUT gas::*chloride* TO Injection TOP SIDE
forget
```

```sfml
INPUT fe:: FROM Power TOP SIDE
OUTPUT fe:: TO Overclocker
forget
```

Non-item types MUST be explicit on both INPUT and OUTPUT (§7).

#### 3.7 EACH source (fan-in)

`EACH` before a label pulls from every block carrying that label
(codebook-attested; local files use `EACH` on OUTPUT, same position rule):

```sfml
INPUT *Shard* FROM EACH Grinder
OUTPUT *Shard* TO Buffer
forget
```

#### 3.8 Side postfix

```sfml
INPUT *Shard* FROM Injection RIGHT SIDE
OUTPUT *Shard* TO EMPTY SLOTS IN Shards
forget
```

```sfml
INPUT fe:: FROM Power TOP SIDE
OUTPUT fe:: TO Injection BACK SIDE
forget
```

Pattern: `FROM <label> <SIDE> SIDE` / `TO <label> <SIDE> SIDE`. Full side
vocabulary in §6.

#### 3.9 SLOTS restriction

```sfml
INPUT *Honey* FROM Honeycombs
OUTPUT *Honey* TO HoneyCentrifuge SLOTS 9-11
forget
```

```sfml
INPUT FROM Centrifuge SLOTS 0-8
OUTPUT TO Storage
forget
```

`SLOTS n-m` (inclusive range) applies to the adjacent label only — source
side in the second snippet, destination side in the first.

#### 3.10 Tagged WITH / WITHOUT [version-dependent]

Per the ATM10 reference: `WITH (...)` / `WITHOUT (...)` constrains by item
tags/components. NOT observed locally; syntax varies by SFM version — verify
against `SFML.g4` before use:

```sfml
-- [INFERENCE — confirm tag syntax against SFML.g4; version-dependent]
INPUT item::minecraft:wool WITH (minecraft:wool_color) FROM Chest
OUTPUT item::minecraft:wool TO Storage
forget
```

### 4. OUTPUT reference

General shape:

```sfml
OUTPUT [count] [RETAIN n [EACH]] [filter] TO [EACH] <label>[, ...] [<SIDE> SIDE] [SLOTS n-m] [EMPTY SLOTS IN ...]
```

#### 4.1 Bare output (paired with filtered/typed input)

```sfml
INPUT FROM HoneyCentrifuge SLOTS 0-8
OUTPUT TO Storage
forget
```

Type flows from the INPUT line; a bare `OUTPUT TO` moves whatever the INPUT
matched.

#### 4.2 Filtered output

```sfml
INPUT *comb* FROM Buffer
OUTPUT *comb* TO EMPTY SLOTS IN Import
forget
```

```sfml
INPUT wax* FROM Resupply
OUTPUT wax* TO Gearbox
forget
```

#### 4.3 Count caps

Plain count limits items moved per operation (codebook-attested):

```sfml
INPUT cobblestone FROM Quarry
OUTPUT 64 cobblestone TO Storage
forget
```

#### 4.4 RETAIN stock-to-level

`RETAIN n` keeps `n` units back per matched stack; `RETAIN n EACH` keeps `n`
of each distinct match:

```sfml
INPUT *Raw* FROM Interface
OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
```

```sfml
INPUT *Dirty* FROM DirtyBuffer
OUTPUT RETAIN 36 EACH *Dirty* TO EMPTY SLOTS IN EACH Enrichment LEFT SIDE
forget
```

`18`/`36` here are per-machine stocking levels for the Mekanism chain.

#### 4.5 Multi-label destination `a,b`

Comma-separated labels send to any/all of them (codebook-attested):

```sfml
INPUT iron_ingot FROM Furnace
OUTPUT iron_ingot TO Storage, Overflow
forget
```

#### 4.6 EACH broadcast

```sfml
INPUT *Raw* FROM Interface
OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
```

`TO EACH <label>` addresses every block carrying the label (all Mekanism
injection chambers at once).

#### 4.7 EMPTY SLOTS IN

Directs inserts at empty slots only (avoids scattering across partial
stacks / wrong machine slots):

```sfml
INPUT *Shard* FROM Injection RIGHT SIDE
OUTPUT *Shard* TO EMPTY SLOTS IN Shards
forget
```

```sfml
INPUT *comb* FROM Honeycombs
OUTPUT *comb* TO EMPTY SLOTS IN ImportHoney
forget
```

Combines with `EACH`, `RETAIN`, and sides:

```sfml
INPUT *Clump* FROM Purification RIGHT SIDE
OUTPUT RETAIN 18 EACH *Clump* TO EMPTY SLOTS IN EACH Crusher LEFT SIDE
forget
```

#### 4.8 Side / slots on output

```sfml
OUTPUT gas::*oxygen* TO Purification TOP SIDE
```

```sfml
OUTPUT *Honey* TO HoneyCentrifuge SLOTS 9-11
```

Sides select the cable face; `SLOTS n-m` selects destination slots.

#### 4.9 Overflow chaining via sequential outputs

Multiple OUTPUT lines under one INPUT execute in order — first match wins
per unit, remainder falls through to the next line:

```sfml
INPUT *Honey* FROM Honeycombs
OUTPUT *Honey* TO EMPTY SLOTS IN ImportHoney
forget
INPUT *Honey* FROM Honeycombs
OUTPUT *Honey* TO HoneyCentrifuge SLOTS 9-11
forget
```

First op tops up the import buffer; the second op fills exact centrifuge
slots with whatever is left. Same idiom for gas primary+overflow (§9.7).

### 5. Labels

Labels are code identifiers referenced by `FROM` / `TO`. Observed labels:
`Interface`, `Injection`, `Shards`, `Purification`, `Crusher`, `DirtyBuffer`,
`Enrichment`, `Storage`, `Power`, `Hive`, `Resupply`, `Overclocker`,
`Treater`, `TreaterB`, `BufferStorage`, `Trash`, `Honeycombs`, `ImportHoney`,
`HoneyCentrifuge`, `Centrifuge`, `Buffer`, `Import`, `Gearbox`, `Heater`.

- One block, many labels: a single block can carry several labels and be addressed by any of them.
- One label, many blocks: `EACH` targets exploit this — every Mekanism
  chamber of one stage shares one label (`Injection`, `Purification`,
  `Crusher`, `Enrichment`).
- Dedicated label per machine/role (recommended, matches all local files):
  `HoneyCentrifuge` vs `Centrifuge`, `Treater` vs `TreaterB`,
  `Overclocker`/`Gearbox`/`Heater` each get their own label so module
  resupply lines (`electrode*`, `wax*`) cannot cross-feed.

### 6. Sides

Observed: `TOP`, `LEFT`, `RIGHT`, `BACK`. Full vocabulary per codebook and
grammar: `TOP`, `BOTTOM`, `LEFT`, `RIGHT`, `FRONT`, `BACK`, compass sides,
and `EACH SIDE`:

```sfml
INPUT fe:: FROM Power TOP SIDE
OUTPUT fe:: TO Injection BACK SIDE
OUTPUT fe:: TO Purification BACK SIDE
OUTPUT fe:: TO Crusher BACK SIDE
OUTPUT fe:: TO Enrichment BACK SIDE
forget
```

```sfml
INPUT *Shard* FROM Injection RIGHT SIDE
OUTPUT *Shard* TO EMPTY SLOTS IN Shards
forget
```

Notes:

- Syntax is always `<SIDE> SIDE` (`LEFT SIDE`, `TOP SIDE`); `EACH SIDE`
  addresses all faces (codebook-attested).
- Compass sides (`NORTH`/`SOUTH`/`EAST`/`WEST`) refer to world directions;
  the rest are relative to the cable connection. [INFERENCE: relative-frame
  details are version-dependent; confirm against SFML.g4.]
- The SFM side MUST hit a face whose slot mode permits the operation
  (local chain feeds `LEFT SIDE`, extracts `RIGHT SIDE`, gases `TOP SIDE`,
  energy `BACK SIDE`).

### 7. Resource types

| Prefix | What it matches | Unit / note |
|---|---|---|
| (none) / `item::` | Items | Count = items. Bare ids (`gem`) default to items. |
| `fluid::` | Fluids | Millibuckets (mB); bare `fluid::` = all fluids. |
| `gas::` | Mekanism gases | Bare `gas::` unattested; always qualify (`gas::*oxygen*`). |
| `fe::` / `energy::` | Forge Energy | Bare `fe::` = all FE; moved `EVERY TICK` locally. |

Rules:

- Item is the default: `INPUT gem ...`, `INPUT *Raw* ...` need no prefix.
- Non-item types MUST be explicit on BOTH sides of the op:

```sfml
INPUT fluid:: FROM Hive
OUTPUT fluid:: TO Storage
forget
```

```sfml
INPUT fe:: FROM Power TOP SIDE
OUTPUT fe:: TO Overclocker
OUTPUT fe:: TO Heater
forget
```

- Mixing types in one op is not supported [INFERENCE]: one INPUT/OUTPUT pair
  moves one resource type.

### 8. IF statement skeleton [version-dependent]

`INPUT`/`OUTPUT`/`IF` are the three statements. Local files use no
`IF`; skeleton and keywords below are per the grammar/codebook references —
condition syntax varies by SFM version, so confirm against `SFML.g4`:

```sfml
EVERY 20 TICKS DO
    IF <condition> THEN
        INPUT cobblestone FROM Quarry
        OUTPUT cobblestone TO Storage
    forget
    END
END
```

Attested keyword pool for conditions/branches: comparisons `gt ge lt le eq`
(or symbolic equivalents), logic `and or not`, plus `has`, `else`, `true`,
`false`. Minimal shapes:

```sfml
IF <label> HAS <filter> THEN
END
```

```sfml
IF <condition> THEN
ELSE
END
```

Caveats: exact operand order (`HAS` vs comparison form), whether `forget`
is required inside branches, and nesting depth are version-dependent.
Keep `IF` bodies to one INPUT/OUTPUT pair per branch until verified.

### 9. Pattern catalogue

Each pattern names the snippet shape to copy. All snippets are adapted from
local-file idioms.

#### 9.1 Move-all (drain a machine)

```sfml
INPUT FROM HoneyCentrifuge SLOTS 0-8
OUTPUT TO Storage
forget
```

Use for output-slot dumps. Scope `SLOTS` so upgrade/module slots are never
touched.

#### 9.2 Fan-in (many machines → one buffer)

```sfml
INPUT *Clump* FROM Purification RIGHT SIDE
OUTPUT RETAIN 18 EACH *Clump* TO EMPTY SLOTS IN EACH Crusher LEFT SIDE
forget
```

`EACH` on the destination fans one buffer out; `EACH` on the source
(§3.7) funnels many machines into one buffer. Combine with `RETAIN`
(§9.8) to leave working stock behind.

#### 9.3 Split-by-filter (one source → typed destinations)

```sfml
INPUT FROM Hive
OUTPUT *Honey* TO Storage
OUTPUT *comb* TO BufferStorage
OUTPUT EXCEPT *comb*,gem TO Storage
forget
```

Order matters: specific filters first, `EXCEPT` catch-all last.

#### 9.4 Multi-op-per-tick (stage isolation)

One `forget`-terminated op per pipeline stage inside a single trigger —
the Mekanism chain pattern:

```sfml
INPUT *Shard* FROM Shards
OUTPUT RETAIN 18 EACH *Shard* TO EMPTY SLOTS IN EACH Purification LEFT SIDE
forget
INPUT *Clump* FROM Purification RIGHT SIDE
OUTPUT RETAIN 18 EACH *Clump* TO EMPTY SLOTS IN EACH Crusher LEFT SIDE
forget
```

Each op sees the result of the previous one within the same trigger fire.

#### 9.5 Machine fill-by-side

Feed one face, extract the opposite, utilities on top/back:

```sfml
INPUT *Shard* FROM Shards
OUTPUT RETAIN 18 EACH *Shard* TO EMPTY SLOTS IN EACH Purification LEFT SIDE
forget
INPUT *Clump* FROM Purification RIGHT SIDE
OUTPUT RETAIN 18 EACH *Clump* TO EMPTY SLOTS IN EACH Crusher LEFT SIDE
forget
INPUT gas::*oxygen* FROM Interface
OUTPUT gas::*oxygen* TO Purification TOP SIDE
forget
```

Feed `LEFT`, extract `RIGHT`, gas `TOP`, energy `BACK` — mirrors the local
Mekanism layout and keeps Mekanism slot configs uniform per face.

#### 9.6 Energy broadcast (EVERY TICK)

```sfml
EVERY TICK DO
    INPUT fe:: FROM Power TOP SIDE
    OUTPUT fe:: TO Injection BACK SIDE
    OUTPUT fe:: TO Purification BACK SIDE
    OUTPUT fe:: TO Crusher BACK SIDE
    OUTPUT fe:: TO Enrichment BACK SIDE
forget
END
```

One INPUT, N OUTPUT lines. Separate energy into its own `EVERY TICK` block;
bulk items stay on pulse/20-tick triggers.

#### 9.7 Gas primary + overflow

Same overflow-chaining shape as §4.9, applied to gases:

```sfml
INPUT gas::*oxygen* FROM Interface
OUTPUT gas::*oxygen* TO Purification TOP SIDE
forget
INPUT gas::*oxygen* FROM Interface
OUTPUT gas::*oxygen* TO OverflowTank TOP SIDE
forget
```

First op satisfies the primary consumer; leftovers fall through to the
overflow destination. ([INFERENCE: second label invented for the pattern;
replace with a real overflow label.])

#### 9.8 Rate cap / stock-to-level

```sfml
INPUT *Dirty* FROM DirtyBuffer
OUTPUT RETAIN 36 EACH *Dirty* TO EMPTY SLOTS IN EACH Enrichment LEFT SIDE
forget
```

`RETAIN n EACH` caps how much each destination accumulates per op —
the local rate limiter. Plain counts (§4.3) cap moved amount instead.

#### 9.9 Deny-list sort

```sfml
INPUT gem FROM Hive
OUTPUT gem TO Trash
forget
INPUT FROM Hive
OUTPUT EXCEPT *comb*,gem TO Storage
forget
```

Trash the junk explicitly first, then `EXCEPT`-drain the rest. Order is
load-bearing.

#### 9.10 Broadcast (one source → all labelled)

```sfml
INPUT *Raw* FROM Interface
OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
```

Identical stock to every machine sharing the label in one op.

#### 9.11 Round robin by block/label [INFERENCE]

`TO EACH <label>` distributes across all blocks of the label, but exact
rotation order (round-robin vs fill-first-available) is version-dependent
and NOT pinned by local files — do not rely on strict alternation:

```sfml
INPUT cobblestone FROM Quarry
OUTPUT cobblestone TO EACH Storage
forget
```

For strict one-per-machine cycling, use one op per destination block/label
(§9.4 shape) instead of a single `EACH` broadcast.
---

## Part C — Modifying Programs, Local Repo Walkthrough, Pitfalls, Resources

> Scope: this part covers the code edit loop, the three `.sfml` programs in this repo, code-only failure modes, and where to look next. Setup/hardware, trigger/execution model → Part A. Full syntax + patterns → Part B.

### 1. Modifying workflow: code edit loop

1. **Edit** — change the `.sfml` source directly.
2. **Keep `--` group comments** marking each op group (stage chain, gas, energy, extraction, resupply).
3. **Re-pull labels after rename** — after renaming or adding a label, update every `FROM` / `TO` line referencing it so the code stays consistent.
4. **One op per `forget` block** — never place two `INPUT` / `OUTPUT` pairs in one scope without `forget` between them (see §3).

> Trigger hygiene: program = ordered triggers; each trigger clears its input list after executing — inputs do not leak across triggers. But **within** one trigger, marks persist across ops until `forget` — see §3.

Tooling:

- SFMHub editor (`sfmhub.site/code-editor`) — browser SFML editor.
- VS Code extension (`TeamDman.super-factory-manager-language`) — highlighting, hover docs, snippets, save-time checks, `.sfm` / `.sfml` associations.

### 2. Local repo walkthrough

Three programs, all `NAME`-headed. Conventions shared: `EVERY redstone pulse` for item/gas work (pulse-driven stage chain), `EVERY TICK` / `EVERY 20 TICKS` for energy (and hive extraction). Keywords case-insensitive; statement terminator is `forget` after every op.

#### 2.1 `Mekanism4xOreProcessing.sfml` — Mekanism 4x stage chain

```sfml
NAME "Mekanism 4x Ore Processing"


EVERY redstone pulse DO
--Raw -> Shard -> Clump -> Dirty Dust -> Dust
forget
    INPUT *Raw* FROM Interface
    OUTPUT RETAIN 18 EACH *Raw* TO EACH Injection LEFT SIDE
forget
    INPUT *Shard* FROM Injection RIGHT SIDE
    OUTPUT *Shard* TO EMPTY SLOTS IN Shards
forget
    INPUT *Shard* FROM Shards
    OUTPUT RETAIN 18 EACH *Shard* TO EMPTY SLOTS IN EACH Purification LEFT SIDE
forget
    INPUT *Clump* FROM Purification RIGHT SIDE
    OUTPUT RETAIN 18 EACH *Clump* TO EMPTY SLOTS IN EACH Crusher LEFT SIDE
forget
    INPUT *Dirty* FROM Crusher RIGHT SIDE
    OUTPUT *Dirty* TO DirtyBuffer
forget
    INPUT *Dirty* FROM DirtyBuffer
    OUTPUT RETAIN 36 EACH *Dirty* TO EMPTY SLOTS IN EACH Enrichment LEFT SIDE
forget
    INPUT *Dust* FROM Enrichment RIGHT SIDE
    OUTPUT *Dust* TO Storage
forget

--GAS
    INPUT gas::*oxygen* FROM Interface
    OUTPUT gas::*oxygen* TO Purification TOP SIDE
forget
    INPUT gas::*chloride* FROM Interface
    OUTPUT gas::*chloride* TO Injection TOP SIDE
END

--Energy
EVERY TICK DO
    INPUT fe:: FROM Power TOP SIDE
    OUTPUT fe:: TO Injection BACK SIDE
    OUTPUT fe:: TO Purification BACK SIDE
    OUTPUT fe:: TO Crusher BACK SIDE
    OUTPUT fe:: TO Enrichment BACK SIDE
END
```

What it does:

- **Stage chain, one op per `forget` block:** `Interface` → `Injection` (Raw→Shard) → `Shards` buffer → `Purification` (Shard→Clump) → `Crusher` (Clump→Dirty) → `DirtyBuffer` → `Enrichment` (Dirty→Dust) → `Storage`. Each stage is INPUT (product side) → OUTPUT (next stage input side).
- **`RETAIN` staging counts:** `RETAIN 18 EACH *Raw*` (Injection feed), `RETAIN 18 EACH *Shard*`, `RETAIN 18 EACH *Clump*`, `RETAIN 36 EACH *Dirty*` (Enrichment feed — double). Keeps a working stock at each machine; only the surplus moves. Final `*Dust*` op has no `RETAIN` — everything drains to `Storage`.
- **`EACH` broadcast:** `TO EACH Injection LEFT SIDE` fans out across all blocks sharing the label; machine outputs read from `RIGHT SIDE` while feeds enter `LEFT SIDE`.
- **Gas feeds:** `gas::*oxygen*` → `Purification TOP SIDE`; `gas::*chloride*` → `Injection TOP SIDE`. Both sourced from `Interface`.
- **Energy:** `EVERY TICK DO` block broadcasts `fe::` from `Power TOP SIDE` to `Injection / Purification / Crusher / Enrichment BACK SIDE`.

To modify:

- Add a machine by copying one `forget` triple (INPUT product / OUTPUT next-stage with `RETAIN`).
- Change `RETAIN` counts to tune buffering.
- Add a gas by copying a `--GAS` triple with the correct `gas::*<id>*` filter.
- After renaming any label, update every `FROM` / `TO` line referencing it.

#### 2.2 `ModularBeesHive.sfml` — hive extraction + module resupply

```sfml
NAME "Modular Bees Hive"
--Modified from BucketSt's Modular Hive

EVERY 20 TICKS DO
--Extracting from the Hive
    INPUT FROM Hive
    OUTPUT *Honey* TO Storage
    OUTPUT *comb* TO BufferStorage
    OUTPUT EXCEPT *comb*,gem TO Storage
forget
--Extracting honey from the Hive
    INPUT fluid:: FROM Hive
    OUTPUT fluid:: TO Storage
forget

--FOR WANNABEES
    INPUT gem FROM Hive
    OUTPUT gem TO Trash
forget

--Resupplying the Hive Modules
    INPUT electrode* FROM Resupply
    OUTPUT electrode* TO Overclocker
forget

--For Soul Treats
    INPUT *Jelly FROM Resupply
    OUTPUT *Jelly TO TreaterB

--For Honey (for treats)
    INPUT honey* FROM Resupply
    OUTPUT honey* TO Treater
END

EVERY TICK DO
    INPUT fe:: FROM Power TOP SIDE
    OUTPUT fe:: TO Overclocker
END
```

What it does:

- **20-tick cadence:** `EVERY 20 TICKS DO` — hive products accumulate between runs.
- **Split-by-filter fan-out:** one bare `INPUT FROM Hive` (all items), then three prioritised outputs — `*Honey*` → `Storage`, `*comb*` → `BufferStorage` (feeds the centrifuge program), `EXCEPT *comb*,gem` → `Storage`. Order matters: honey and comb are claimed first, remainder falls through.
- **Fluid drain:** `INPUT fluid:: FROM Hive` → `OUTPUT fluid:: TO Storage`.
- **Junk void:** `INPUT gem FROM Hive` → `OUTPUT gem TO Trash`.
- **Module resupply (reverse direction):** `electrode*` from `Resupply` → `Overclocker`; `*Jelly` → `TreaterB`; `honey*` → `Treater`. Note the last two ops share one `forget` scope — marks accumulate, so the second INPUT adds to the first's remainder [INFERENCE: intentional here since filters are disjoint; splitting with `forget` is safer if filters ever overlap].
- **Energy:** `EVERY TICK DO`, `fe::` from `Power TOP SIDE` → `Overclocker`.

To modify:

- Change the comb destination label to repoint the hive→centrifuge link.
- Add a product line by inserting an `OUTPUT <filter> TO <label>` before the `EXCEPT` fall-through.
- Widen `electrode*` / `wax*`-style resupply ops for new module types.

#### 2.3 `ModularBeesCentrifuge.sfml` — comb routing across two centrifuges

```sfml
NAME "Modular Bees Centrifuge"
--Modified from BucketSt

EVERY redstone pulse DO

--INPUT (2 centrifuges, one without heater to produce wax)
forget
    INPUT *Honey* FROM Honeycombs
    OUTPUT *Honey* TO EMPTY SLOTS IN ImportHoney
forget
    INPUT *comb* FROM Honeycombs
    OUTPUT *comb* TO EMPTY SLOTS IN ImportHoney
forget
    INPUT *Honey* FROM Honeycombs
    OUTPUT *Honey* TO HoneyCentrifuge SLOTS 9-11
forget
    INPUT *comb* FROM Buffer
    OUTPUT *comb* TO EMPTY SLOTS IN Import
forget
    INPUT *comb* FROM Buffer
    OUTPUT *comb* TO Centrifuge SLOTS 9-11
forget
--OUTPUT
    INPUT fluid:: FROM HoneyCentrifuge
    OUTPUT fluid:: TO Storage
forget
    INPUT FROM HoneyCentrifuge SLOTS 0-8
    OUTPUT TO Storage
forget
    INPUT fluid:: FROM Centrifuge
    OUTPUT fluid:: TO Storage
forget
    INPUT FROM Centrifuge SLOTS 0-8
    OUTPUT TO Storage
forget
--Resupplying Centrifuge Modules
    INPUT electrode* FROM Resupply
    OUTPUT electrode* TO Overclocker
forget
    INPUT wax* FROM Resupply
    OUTPUT wax* TO Gearbox
END

EVERY TICK DO
    INPUT fe:: FROM Power TOP SIDE
    OUTPUT fe:: TO Overclocker
    OUTPUT fe:: TO Heater
END
```

What it does:

- **Dual centrifuges:** `HoneyCentrifuge` (no heater, produces wax) and `Centrifuge` (heated). Inputs route `*Honey*` / `*comb*` from `Honeycombs` and `Buffer` labels.
- **Slot-targeted inserts:** `EMPTY SLOTS IN ImportHoney` / `EMPTY SLOTS IN Import` (any free input slot) plus direct `SLOTS 9-11` processing-slot fills on each centrifuge. `EMPTY SLOTS` avoids overwriting in-progress stacks.
- **Slot-ranged drains:** `INPUT FROM HoneyCentrifuge SLOTS 0-8` (bare filter = all items in those slots) → `Storage`; same for `Centrifuge SLOTS 0-8`. Fluids drain separately via `INPUT fluid:: FROM <centrifuge>` → `Storage`.
- **Module resupply:** `electrode*` → `Overclocker`, `wax*` → `Gearbox`, both from `Resupply`.
- **Energy:** `EVERY TICK DO`, `fe::` → `Overclocker` + `Heater`.

To modify:

- To add a third centrifuge, copy one INPUT triple (source label + `SLOTS 9-11` target) and one OUTPUT pair (fluid + `SLOTS 0-8`).
- To change slot maps, edit the `SLOTS n-m` ranges.
- To switch wax/heater behaviour, move the `Heater` FE output line.

### 3. Pitfalls / troubleshooting

| # | Symptom | Cause | Fix |
|---|---------|-------|-----|
| 1 | Later ops move wrong items; cascade grows each run | Missing `forget` — marks leak into the next op within the same trigger | `forget` after **every** op. Habit: never two INPUT/OUTPUT pairs without `forget` between them (the `*Jelly` / `honey*` shared-scope pair in §2.2 is the risky exception) |
| 2 | Machine never fills, no error | Unsided IO — that target requires explicit sides | Always suffix sides on sided labels: `LEFT SIDE` feed, `RIGHT SIDE` drain, `TOP SIDE` gas, `BACK SIDE` energy. Bare `TO Purification` silently fails |
| 3 | Machines starve for power despite FE source | Per-tick source transfer limit; single-side drain caps throughput | Bigger buffer + drain `each side` / feed all consumer sides |
| 4 | Parse error on item id (`redstone`, etc.) | Id collides with SFM keyword | Quote or namespace: `item:minecraft:redstone`, `item::redstone`, or quoted form. Same for any id matching `redstone`, `side`, `each`, … |
| 5 | Some marked resources never move, rest fine | Overlapping marks silently ignored | Keep filters disjoint per op, or order ops so the priority claim comes first (hive `*Honey*` before `EXCEPT` fall-through). Split ambiguous ops with `forget` and re-INPUT |
| 6 | `Conditions` / `IF` syntax unclear | Codebook `Conditions` section is a TODO stub | Treat `IF ... THEN ... END` skeletons as [INFERENCE] until checked against `SFML.g4` grammar |

### 4. Resources (ranked)

Tier 1 — authoritative; cite as ground truth.

- Official repo — https://github.com/TeamDman/SuperFactoryManager — source, issues/Discussions, releases, `template_programs/`, GameTests. Wiki is empty — do not cite it.
- SFML grammar — https://github.com/TeamDman/SuperFactoryManager/blob/1.19.2/platform/minecraft/src/main/antlr/sfml/SFML.g4 (path varies by branch) — final authority on conflicts.
- CurseForge — https://www.curseforge.com/minecraft/mc-mods/super-factory-manager — downloads, MC × loader matrix, changelog. Cross-check via GitHub releases.
- Modrinth — https://modrinth.com/mod/super-factory-manager — mirror downloads (MC 1.19.2 → 26.1.2). Use for versions/links, not prose.
- VS Code extension — https://marketplace.visualstudio.com/items?itemName=TeamDman.super-factory-manager-language — highlighting, hover docs, snippets, save-time checks, `.sfm` / `.sfml` associations.
- Discord — https://discord.gg/5mbUY3mu6m — live help, version-specific answers.
- SFMHub guide — https://sfmhub.site/guide — Examples / Getting Started / Basics tabs; write-code-first workflow; trigger + 3-statement model; changelog (SFM 4.19–4.28 on MC 1.21.1).

Tier 2 — language references.

- monotoast SFML manual (SFM 4.34) — https://monotoast.github.io/sfm-guide/ — triggers, `forget` semantics, resource IDs, EACH table. Cross-check against grammar/GameTests.
- Revenantal ATM10 `SFM_REFERENCE.md` — https://github.com/Revenantal/atm10-sfm-scripts/blob/main/SFM_REFERENCE.md — 511-line working reference (retain semantics, EACH, wildcards vs regex, WITH/WITHOUT tags, sides/slots forms).
- Official examples dir — https://github.com/TeamDman/SuperFactoryManager/tree/1.19.2/examples — `01-moveitems` … `08-fluids.sfm` canonical minimal samples.

Tier 3 — copy-paste corpora (mine patterns, verify versions).

- schroenser programs — https://github.com/schroenser/super-factory-manager-programs — Mekanism (antimatter, fissile fuel, fusion, HDPE, plutonium, polonium, electrolysis), bees, HNN, FTB Skies 2.
- LuisM360 examples — https://github.com/LuisM360/super-factory-manager-examples — HNN + Mekanism gas-burning power setups.

Tier 4 — video.

- TheVoos tutorial (MC 1.21) — https://www.youtube.com/watch?v=esz5FVA-fY0 — first program 2:40, FE 5:43, fluids 9:50, slots 12:07, EXCEPT 14:15, labeling 22:00+, priorities 29:00, round-robin 31:40, bees 35:15.
- LEGS overview — https://www.youtube.com/watch?v=E-g8-gQut8M — auto-storage, HNN, resource gen demo.
- Official spotlight — https://www.youtube.com/watch?v=W5wY23VxZAc — canonical feature overview (repo README banner).

Tier 5 — threads & background.

- Reddit: SFMHub announcement — https://www.reddit.com/r/feedthebeast/comments/1pi8ret/i_made_a_super_factory_manager_hub_to_host/ ; docs-location thread — https://www.reddit.com/r/feedthebeast/comments/1p3540g/the_documentation_for_sfm_is_for_some_reason_at/ ; ATM10 Mekanism/AE2 + troubleshooting threads — failure modes + integration patterns.
- MC百科 (mcmod.cn) — https://www.mcmod.cn/post/4271.html, https://www.mcmod.cn/post/6685.html, class page https://www.mcmod.cn/class/1840.html — Chinese tutorials incl. bees.
- Mekanism ore background (not SFM-specific) — https://wiki.aidancbrady.com/wiki/Ore_Processing — stage order and yield facts behind §2.1.

Excluded (do not cite): `sfm.supremainc.com` Suprema fingerprint-SDK docs (acronym collision); `superfactorymanager.ca` (empty portal at read time); no standalone SFM simulator exists — GameTests + parser/AST reuse are the closest.

### 5. Attribution

- Relvl `mc-sfm-codebook` (single `README.MD`, no `LICENSE` / `NOTICE` at read time): treat as **unlicensed** — learning/paraphrase OK; credit `Relvl/mc-sfm-codebook` on any verbatim copy.
- SFMHub guide content is user-provided; footer disclaims association with Mojang/Microsoft and the SFM mod team — do not attribute guide copy to either.
- `ModularBeesHive.sfml` / `ModularBeesCentrifuge.sfml` note `--Modified from BucketSt` / `BucketSt's Modular Hive` — retain that credit when redistributing those programs.

