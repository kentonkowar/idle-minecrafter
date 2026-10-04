# Design: Measuring Rates

2026-09-30 · smithy · Status: Draft for review

## Problem

The PID requires offline rates to come from what a setup actually does while the player is present. It also requires that offline output never exceeds what the setup would have produced online. So the mod must watch each registered setup and turn what it sees into a fair rate. Four things make that hard:

- **Deposits.** Output storage is an ordinary container. The player's own hoppers can fill it with items the setup didn't make, and they can drain it while the setup runs.
- **Change.** A farm that has been rebuilt, or whose villagers have died, no longer produces at its old rate.
- **Randomness.** Mob farms are random, and many machines work in bursts, so any short measurement is noisy.
- **Running time.** A rate only means something for the time the setup was loaded and running.

This doc defines what counts as production, how it is counted, when counting starts over, and how a noisy count becomes a rate that meets the 5% fairness goal. It produces a **rate card** for each setup, which Catch-up Calculation uses.

## Chosen approach

The mod counts events as they happen; it never takes snapshots of storage contents. An item counts as output only when both of these are true:

1. **Generated:** the item came into existence inside the setup's region, from a cause its type allows (a mob dying, a machine breaking a block, a crafter crafting). It did not come from a player.
2. **Arrived:** automation inserted the item into one of the setup's Output storage blocks. It did not come from a player's hand.

Counts go into one-minute buckets over a rolling one-hour window, and only the time the setup was running counts. The rate used offline is the lower edge of a confidence range, not the average. That way, random variation makes offline output fall slightly short of online output rather than exceed it.

### What counts as generated

| Type     | Generated                                                                                                                                 | Not generated                                                                                                     |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Mob farm | Loot from mobs that die in the region, and items that mobs produce there (laid eggs, wool sheared by dispensers, shed scutes)             | Loot from a mob a player hurt in the 5 seconds before it died. This is the game's own rule for player-kill drops. |
| Quarry   | Drops from blocks in the region broken by something other than a player: explosions, pistons pushing breakable blocks, and flowing fluids | Blocks a player mined, and explosions a player ignited                                                            |
| Crafting | Results from crafters, furnaces, smokers, blast furnaces, composters, and brewing stands in the region                                    | Items a player crafts by hand                                                                                     |

For every type, these items are never counted as generated:

- items a player throws
- items that droppers or dispensers eject
- the contents of a container that breaks
- anything that enters the region already as an item
- inventories the player opens

Each generated item adds one **credit** for its item type to the setup. A credit expires after 5 minutes (configurable). That is the longest an item may take to travel from where it was made to Output storage.

### What counts as arrived

An arrival is an item inserted into one of the setup's Output storage blocks by a hopper, hopper minecart or dropper. Items a player inserts through a screen never count. Taking items out never affects the count.

Each arrival uses up one credit of the same item type. An arrival with no credit left is treated as a deposit from the player's own system, and it is not counted.

**Example:** an iron farm generates 600 iron ingots in an hour. The player also pipes 200 ingots from a trading hall into the same hopper line. Only 600 credits exist, so at most 600 ingots are counted, however the two streams mix.

**Residual case:** if some of the farm's own items are lost (burned or despawned), items the player deposits can take their place. They can never push the count above what the region generated.

Arrivals are counted by insertion, not by looking at contents. So the method works the same for Idle Storage and for linked tagged storage. For linked storage, only vanilla hoppers, hopper minecarts and droppers are seen; insertions by other mods' pipes are not counted, since machines from other mods are out of scope.

### What counts as consumed

Crafting setups, and mob farms with inputs, use items up. Consumption is counted where the items are used inside the region, not where they leave Input storage:

- the ingredients a crafter uses
- the input and fuel a furnace, smoker or blast furnace uses
- the ingredients a brewing stand uses
- items a dispenser uses up, such as thrown eggs

It doesn't matter where these items came from. Offline, all consumption is taken from Input storage. So a machine the player feeds by hand while online can't produce for free while offline.

### When a setup counts as running

Only running ticks count, and events during any other tick are ignored.

| Condition | Rule |
| --- | --- |
| Loaded | Mob farm and crafting: every chunk in the region is ticking entities. Quarry: at least one chunk is ticking, because the machine moves and a 64-chunk region can't be loaded all at once. |
| Not blocked | No Output storage is full. |
| Not starved | For crafting setups and mob farms with inputs: every item the setup has been consuming is still present in its Input storage. |

The mod also records how many running ticks passed per real second. That tick rate is used when offline time is converted to ticks, and it is capped at 20. So a world that lagged online gets less offline output, and speeding up ticks online earns nothing extra.

### Buckets and the window

- **Bucket:** 1,200 running ticks (one minute at normal speed). A bucket stores the running ticks, the count of each output item, the count of each consumed item, and, for quarries, the number of blocks removed.
- **Window:** the latest 60 buckets, which is one hour of running time. When a new bucket starts, the oldest one is dropped. An hour covers three full day-night cycles, so farms that rely on darkness are averaged fairly.
- **Minimum:** until the window holds 10 buckets, the setup is **Measuring** and produces nothing offline.
- **Saved:** the window is saved with the setup, so measurement carries on when the player logs off and back in.

### Handling randomness

For each output item:

> offline rate = mean − 1.28 × standard error

- The **mean** is the item's count per running tick over the window.
- The **standard error** comes from how much the per-bucket counts vary. The mod doesn't assume production is smooth. So a machine that makes a batch every few minutes automatically gets a wider margin than a steady one.
- The factor **1.28** gives a 90% chance that the offline rate is at or below the setup's true average rate. The rate never goes below zero.

Consumption uses the plain mean per running tick, with no margin. Output is slightly understated while inputs are not, so offline crafting never gets its outputs more cheaply than online crafting.

For a steady farm that counts N items of one kind in the window, the expected shortfall is about 1.28 ÷ √N:

| Items per hour | Expected shortfall |
| --- | --- |
| 100 | 12.8% |
| 250 | 8.1% |
| 655 | 5.0% |
| 1,000 | 4.0% |
| 5,000 | 1.8% |

Any item made at fewer than about 650 per hour misses the 5% goal with a one-hour window. The shortfall shrinks with the square root of the window's length, so a two-hour window (a config option) meets the goal down to about 330 per hour. The controller screen shows each item's rate and its margin, so the player can see how accurate the measurement is.

### Reset conditions

A **reset** clears the window, and the setup goes back to Measuring.

| Event | Effect |
| --- | --- |
| The setup is confirmed again after any edit (region, type, storage, links, joint chunks) | Reset |
| The setup is repaired after being Invalid | Reset |
| A player places or breaks a block in the region or its joint chunk | Reset, except blocks in `#idleminecrafter:ignored_by_measurement` (signs and banners by default) |
| A world setting that affects production changes: difficulty, or the game rules for mob spawning, mob loot, block drops, mob griefing or random tick speed | Every setup resets |
| Measurement settings change in the config | Every setup resets |
| The setup changes its own region (machines, explosions, pistons, fluids, mobs) | No reset |
| The player presses buttons or levers, opens containers, takes output or adds input | No reset |
| A storage block is upgraded | No reset |
| The player kills mobs in the farm | No reset; those drops just don't count |

Player block changes reset the whole window instead of waiting for it to adapt. Otherwise a player could take a farm apart and log off, and 59 of the window's 60 minutes would still show the old rate.

**Stalled:** a setup that is quietly broken (its villagers died, or a mob got stuck) produces nothing, but no player touched a block. So if a rated setup counts no output for its stall time, it becomes **Stalled**, and its offline rate is zero until output is counted again. The window is kept.

- The stall time is 5 running minutes, or 5 times the longest gap between outputs in the window, whichever is longer.
- Blocked and starved ticks don't count toward the stall time.

### Measurement states

These refine the Registered state from Registering Setups.

| State | Meaning | Offline output |
| --- | --- | --- |
| Measuring | The window holds fewer than 10 buckets | No |
| Rated | The window is full enough, and output is being counted | Yes, from the rate card |
| Stalled | No counted output for the stall time | No |

### Rate card

The rate card is what Catch-up Calculation receives for each setup. It is computed from the window when the world saves. It is saved with the setup in `data/idleminecrafter_setups.dat`.

| Field | Content |
| --- | --- |
| `status` | `MEASURING`, `RATED` or `STALLED` |
| `runningTicks` | Running ticks in the window |
| `tickRate` | Observed running ticks per real second, capped at 20 |
| `outputs` | For each item: offline rate, mean and standard error, per running tick |
| `consumption` | For each item: mean consumed per running tick |
| `quarry` | Blocks removed per running tick (after the confidence margin); for each item, the share of generated drops that arrived; the direction the removal front moves and the cross-section it sweeps |
| `accuracy` | Expected shortfall of the main output |
| `lastReset` | Game time and reason |

Credits are not saved. They only last minutes, and losing them at a save can only make the count lower.

### Where events come from

Every event looks up its chunk in the chunk-to-setup index from Registering Setups. That is one lookup, and events outside any region stop there.

| Event | Vanilla behaviour observed |
| --- | --- |
| Mob loot | A mob dies and drops loot, together with its killer and whether a player hurt it recently |
| Mob products | A chicken lays an egg, a dispenser shears a sheep, a mob sheds a scute |
| Block drops | A block breaks with drops, together with the cause (explosion and who ignited it, piston, fluid, player) |
| Crafts | A crafter crafts; a furnace, smoker or blast furnace finishes an item; a brewing stand finishes |
| Consumption | The ingredients and fuel used by those crafts; a dispenser uses up an item |
| Arrivals | An item is inserted into Output storage, together with the block or entity that inserted it |
| Running | The chunk ticking state and Input and Output storage fill state, checked for registered setups only |

### Commands

These are testing commands, like those in Registering Setups.

| Command | Effect |
| --- | --- |
| `/idle rates <setup>` | Shows the rate card, with each item's rate and margin |
| `/idle measure reset <setup>` | Clears the window |
| `/idle measure export <setup>` | Writes the window's buckets to a CSV file in the world folder, for fairness tests |

## Proving the 5% goal

| Source of error | How it is handled |
| --- | --- |
| Randomness | Lower confidence bound. The shortfall is 5% or less for items counted at about 650 or more per hour. |
| Bursty machines | The standard error comes from the spread between buckets, so bursts widen the margin. |
| Items lost before storage | They never arrive, so they are never counted. |
| Items in transit at the window's edges | At most 5 minutes of transit against a 60-minute window. This affects both ends equally, so it roughly cancels out. |
| Partly loaded regions | The running rules. Any leftover dilution only lowers the rate. |
| Lag or sped-up ticks | The observed tick rate is used, capped at 20. |
| Day and night | The window covers whole day-night cycles. |
| Player-made output | Not generated, so not counted. |

**Fairness test.** There is one automated game test for each type, each with a reference build: a spawner mob farm, an iron farm, a TNT tunnel bore as the quarry, and a crafter line making iron blocks.

1. Each build runs online until it is rated.
2. The build then runs online for 2 more hours, 10 times with different seeds.
3. Catch-up grants 2 offline hours from the rate card.

The test passes when offline output of the main item is at or below the online mean in at least 9 of the 10 runs, and within 5% of that mean.

## Alternatives considered

| Alternative | Why not chosen |
| --- | --- |
| Periodic snapshots of storage contents | Hoppers draining output hide production, and player deposits look like production |
| Count every automated insertion into output | The player could pipe items in from anywhere |
| Count generation only, not arrival | Items that are lost or despawn would count, so offline output would exceed online |
| Mark items with a data component at their source | Changes how items stack, writes mod data into items across the world, and leaves unknown data behind if the mod is removed |
| Work out the rate from the build (spawn spaces, machine design) | The mod would be guessing; the PID requires observed behaviour |
| Average since confirmation, with no window | Never adapts when a farm degrades |
| Use the plain average | Offline output would exceed online output about half the time |
| Reset on any block change in the region | Quarries and farms change their own region all the time |
| Count consumption where items leave Input storage | A machine fed by hand while online would produce for free while offline |

## How it meets the essentials

| Essential | How this design meets it |
| --- | --- |
| No save corruption or item loss; the world loads after removal | Measurement data lives only in the mod's world-data file. Nothing is written to items or blocks. |
| Offline output ≤ online output | Only output that is both generated and arrived counts, and it counts only during running ticks. The rate is a lower bound, not the average. Player changes reset the window, and broken farms stall to zero. |
| No duplication exploits | An arrival needs a credit, so deposits don't count. Measurement only runs while the world ticks, so logging in and out changes nothing. |
| Works with the computer off | Measurement happens while playing, and the rate card is saved for the next load. |
| Calculations on the server | All counting happens on the server. The client only shows the results. |
| Offline time can be capped or disabled | Not affected. The window, margin, credit time and stall time are configurable. |

## Hand-offs to the other design docs

- **Catch-up Calculation:**
  - Offline running ticks are the real time elapsed, limited by the cap, multiplied by `tickRate`.
  - Output is produced at the offline rate. Consumption runs at the mean rate, and production stops when an input it needs runs out.
  - Setups that are Measuring or Stalled produce nothing.
  - A quarry removes blocks at the measured rate, in the measured direction. Its items are those blocks' drops, multiplied by the share of drops that arrived.
- **Delivery & Summary:** show each setup's measurement state and accuracy. When a setup produced nothing, say why: still Measuring (with the minutes left), Stalled, starved, or Invalid.
- **Registering Setups:** the Registered state gains the Measuring, Rated and Stalled states described above. This design can count output in linked tagged storage.

## Open questions

- [ ] Confirm the defaults in the Prototype phase:
  - 1-minute bucket
  - 60-bucket window
  - 10-bucket minimum
  - 1.28 margin factor
  - 5-minute credit
  - 5-minute stall time
- [ ] A crafting setup with independent lines (iron blocks and red dye): when one input runs out offline, should everything stop, or should the mod pair each output with its own inputs?
- [ ] Should the 5% goal apply to a setup's main output only, or to every item it makes?
- [ ] Is experience counted as output? It is not an item, so this design leaves it out of v1.0.
- [ ] How a quarry's progress is modelled offline (to be settled in Catch-up Calculation).
