# Design: Catch-up Calculation

2026-10-02 · smithy · Status: Draft for review

## Problem

When the world loads, the mod must work out what each registered setup would have produced while the player was away. The PID sets the limits:

- Offline output never exceeds online output.
- Offline crafting uses up inputs, and output stops when inputs run out or storage fills.
- Offline time can be capped or switched off.
- Nothing runs while the computer is off, so all the work happens at load.
- Catch-up must finish without a noticeable freeze: under 1 second for 20 setups after 12 hours offline.

Catch-up takes the rate cards from Measuring Rates and the setup definitions from Registering Setups. It produces, for each storage block, the items to add or remove, which Delivery & Summary then applies and reports.

## Chosen approach

Catch-up runs once each time the world starts. It treats every setup as a set of constant rates, and it splits the offline period at **breakpoints**: the moments when a storage block fills or an input runs out. Between two breakpoints, every amount is a rate multiplied by a time, so nothing is replayed tick by tick. Only quarries read chunk data, and they do it off the main thread.

This draft assumes these recommendations from the Measuring Rates review, which are still open:

- Output storage is Idle Storage only in v1.0.
- `tickRate` is the world's ticks per unpaused real second, capped at 20.
- Consumption is scaled with output, so inputs and outputs keep their online ratio.

### When it runs

1. The server starts and the mod's world data loads.
2. Catch-up solves every setup that is not a quarry and does not take items from a quarry. 
3. Quarries read their chunk data in the background. Their results, and the results of any setup downstream of them, are finished when the reads complete. Until then the summary shows them as **Calculating**.

### Offline time

| Value | Rule |
| --- | --- |
| `highestSeen` | The latest real-world time the mod has recorded. It is written at every save and never moves backwards. |
| Offline time | Current time − `highestSeen`, or zero if the clock is earlier than `highestSeen` |
| Granted time | Offline time, limited by `offlineCap` |
| `offlineCap` | Default 2 hours. There is no maximum. 0 turns catch-up off. |
| Offline ticks | Granted time in seconds × the setup's `tickRate` |

Time with the world open never counts, even while the game is paused. Setting the clock forward grants that time once, within the cap. Setting it back grants nothing until real time passes `highestSeen` again.

### Starting state

Catch-up never loads a chunk to find out what is in storage. It starts from saved data:

- **Storage snapshot:** when a chunk that holds Output or Input storage saves, the mod records each block's contents (item counts and free slots) in its world data. Linked input storage is included.
- **Pending items:** items from an earlier catch-up that have not been delivered yet, because the player has not loaded that chunk, count as already in storage.
- **Carried remainders:** the fractions left over from the last catch-up (see Rounding).
- **Rate card** for each setup.

Only **Rated** setups produce. A setup that is Measuring, Stalled, Invalid or Draft produces nothing. It still takes part in chains: as a producer it adds nothing, and as a consumer it takes nothing.

### Production lines

A setup's outputs are split into **lines**, so that one input running out stops only the outputs that need it.

- Measuring Rates records which inputs each generating event used: a crafter's ingredients with its result, a furnace's input and fuel with its result, a brewing stand's ingredients and bottles with its potions, and a composter's inputs with its bone meal.
- Outputs and inputs joined by any such event form one line. An input shared by several recipes, such as furnace fuel, joins their lines into one.
- Outputs with no recorded inputs (mob loot, mob products, quarry drops) form a line with no inputs. In a mob farm that has inputs, the mod cannot tell which input a mob drop needed, so every output depends on every input.
- Items made and used up inside the region never appear as inputs, because Measuring Rates spends their credit instead.
- A line's **main output** is its output with the highest count in the window.

**Example:** an iron farm feeds a crafting setup that makes iron blocks and red dye. The crafting setup has two lines: iron ingots to iron blocks, and poppies to red dye. When the poppies run out, red dye stops and iron blocks continue.

### Rates

For each running line:

- Each output is produced at its offline rate from the rate card.
- Each input is used at its mean rate × (main output's offline rate ÷ main output's mean rate). That keeps the online ratio of inputs to outputs.

### Solving the offline period

The solve works forward from the start of the offline period:

1. Work out the net rate of every item into or out of every storage block, from every running line.
2. From those rates, find the earliest breakpoint:
   - an Output storage block fills for an item
   - an Input storage block runs out of an item
   - a quarry front reaches the edge of its region or an unreadable chunk
   - the granted time ends
3. Move every amount forward to that time, apply the breakpoint's rule (below), and repeat.

Each breakpoint changes one storage block or one line, so there are at most a few per storage block and item. The cost depends on how many setups, lines and storage blocks there are, not on how long the player was away. That matters because the cap has no maximum.

### Chained setups

All setups are solved together in one pass, not one after another. A producer and its consumer therefore run at the same time, as Registering Setups requires.

- The stock in a shared storage block changes at the producer's rate minus the consumer's rate.
- When a shared block runs empty, the consumer's line does not stop. It slows to the rate the producer delivers. It stops only if that rate is zero.
- When a shared block fills, the producer's item overflows as described below.
- Because chains have no loops, rates are worked out from producers to consumers in a single pass at each breakpoint.

### Output storage and overflow

- **Where items go:** each item goes to the Output storage blocks it arrived in online, in the measured shares. Within a storage chain, the furthest block fills first, as Registering Setups describes.
- **Overflow:** when an item's block is full, the item goes to the setup's other Output storage blocks that have space. Blocks that already hold that item come first, then the block with the most free space.
- Overflow never goes into a joint chunk's shared storage, because that would send another setup items the player didn't route to it.
- **All full:** when every Output storage block is full for an item, that item stops.
  - If it is a line's main output, the whole line stops and stops using inputs.
  - If it is any other output, the line continues and that item is not produced.
- **Capacity** is counted in whole stacks: a new stack takes a whole slot, even if it is only partly filled. Delivery then always fits.

### Inputs

- A line takes inputs only from its setup's Input storage, starting from the snapshot plus anything chained producers deliver.
- When an input runs out and nothing refills it, every line that uses it stops. Other lines continue.

### Rounding

- Output totals are rounded down, and input totals are rounded up.
- The fractions left over are saved for each setup and item and added at the next catch-up. Over time nothing is lost, and no single catch-up gives more than was earned.

### Quarries

Quarries use the blocks that are actually in the quarry's path.

- **Front:** the rate card gives the direction, the cross-section and the front's speed (distance per running tick, after the confidence margin). The front advances by speed × offline ticks, and stops at the edge of the region.
- **Reading:** the mod reads the saved data of the chunks the front crosses, off the main thread, without loading them into the world. The front stops at the first chunk that has not been generated yet.
- **Which blocks count:** only the block types the quarry removed while it was being measured. Blocks that hold items or other data, such as containers, are never removed. Every other block in the path stays in place.
- **Items:** each counted block gives the drops from its loot table for the cause the quarry used (explosion, piston or fluid). These are averaged over many rolls for each block type and multiplied by the count. The result is then multiplied by each item's share of drops that arrived online.
- **Storage:** quarry items go through the same solve as other output. If the quarry's Output storage fills, the front stops there, and blocks beyond that point are not removed. A quarry with no Output storage removes blocks and gives no items.
- **Removal:** the cleared volume is recorded for each chunk and applied when that chunk next loads, spread over several ticks.
- **The machine** stays where it was. When the player restarts it, it crosses the cleared space before it reaches new blocks.

### Results

For each setup, catch-up records:

- the granted time and the offline ticks
- items added to each storage block, and items removed from each Input storage block
- items passed to each chained consumer
- when and why each line stopped early (for example, "Output full at 47 min" or "Out of poppies at 1 h 12 min")
- for quarries, the cleared volume per chunk

These records, the new `highestSeen` and the new remainders are saved together. If the game crashes before that save, the next load runs catch-up again from the same starting state, so it is never applied twice.

### Performance

| Part | Cost | Where it runs |
| --- | --- | --- |
| Setups that are not quarries | Grows with setups, lines and storage blocks; not with time offline | Main thread, before the player joins |
| Quarry reads | Grows with how far the front moves, limited by the region size (64 chunks by default) | Background thread |
| Quarry block removal | Spread over ticks as each chunk loads | Main thread, after load |

The 12-hour speed goal in the PID stays as the test case. Because the solve's cost does not depend on time offline, longer caps do not make it slower. Only quarry reads grow, and those are limited by region size.

### Commands

These are for testing and need cheats enabled.

| Command | Effect |
| --- | --- |
| `/idle catchup preview <setup> <hours>` | Runs the solve for that many hours and prints the result without applying it |
| `/idle catchup simulate <hours>` | Applies catch-up to every setup as if the world had been closed for that long. Used by the fairness test. |
| `/idle catchup last <setup>` | Shows the setup's last catch-up record |

## Alternatives considered

| Alternative | Why not chosen |
| --- | --- |
| Replay ticks for the offline time | Far too slow, and it needs every chunk loaded |
| Fixed time steps (for example, one-minute steps) | Cost grows with time offline, and the cap has no maximum |
| Solve each setup on its own, producers first | A producer would be blocked by shared storage that its consumer is draining at the same time |
| Load chunks at start to read storage | Causes a freeze at load. Snapshots give the same information. |
| Quarry items from the measured item mix | Faster, but the mix can differ from the blocks actually in the path |
| Quarry items without removing blocks | The quarry would later mine the same blocks again, which duplicates items |
| Whole setup stops when one storage fills | Unfair to setups whose items are sorted into separate storage |
| Whole setup stops when one input runs out | Unfair to setups with independent lines |
| Round every total down with no carry | A slow item could never be produced |
| Trust the system clock as it is | Setting the clock back and forward would grant the same time again and again |

## How it meets the essentials

| Essential | How this design meets it |
| --- | --- |
| No save corruption or item loss; the world loads after removal | Catch-up writes only to the mod's world data. Quarries never remove containers. Capacity is counted in whole stacks, so delivered items always fit. A line stops using inputs when its main output can't be stored. |
| Offline output ≤ online output | Rates are the lower confidence bound. Output is limited by storage capacity, input stock, the region edge and the cap, and totals round down. |
| No duplication exploits | Offline time comes from `highestSeen`, so clock changes and repeated logins add nothing. Results save in one step. Offline quarries remove the blocks they mined. |
| Works with the computer off | Everything is calculated at load from saved data. |
| Calculations on the server | The whole solve runs on the server. The client only shows the summary. |
| Offline time can be capped or disabled | `offlineCap` limits granted time, and 0 turns catch-up off. |

## Hand-offs to the other design docs

- **Measuring Rates:**
  - Record which inputs each generating event used, so outputs can be grouped into lines. This answers its open question about independent crafting lines.
  - Record each item's share of arrivals per Output storage block.
  - For quarries, measure the front's speed with the confidence margin, the cause it uses, and the block types it removes.
  - Don't mark a quarry Stalled while its front crosses space that catch-up cleared.
- **Registering Setups:** the data model gains the storage snapshots, `highestSeen`, carried remainders, pending items and the latest catch-up record.
- **Delivery & Summary:**
  - Apply pending items to each storage block when its chunk loads.
  - Apply quarry removal when each chunk loads, spread over ticks.
  - Show why each line stopped, what passed along chains, and quarries still Calculating.
  - `/idle prepare-removal` must deliver all pending items first.

## Open questions

Verify these defaults:

- [ ] Overflow order: blocks that already hold the item first, then the most free space. Overflow never goes into joint shared storage.
- [ ] When a line's main output can't be stored, the line stops. When a by-product can't be stored, it is simply not produced.
- [ ] In a mob farm with inputs, every output depends on every input.
- [ ] A quarry's front stops when its storage fills, and at chunks that have not been generated.
- [ ] Quarry drops are averaged over many rolls for each block type.
- [ ] Outputs round down, inputs round up, and the remainders carry forward.
- [ ] The `highestSeen` clock rule.
- [ ] With no maximum cap, the 12-hour speed goal stays as the test case.

Still open:

- [ ] Is experience granted offline? Measuring Rates leaves it out of v1.0.
- [ ] Should quarry results that are still Calculating block the player from opening that quarry's storage?
