# Design: Registering Setups

2026-09-30 · smithy · Status: Draft for review

## Problem

The mod must know exactly which player builds count for offline progress, where each one's output goes, and where its inputs come from. The PID requires that the player chooses (the mod never guesses), that registration can't happen by accident, and that all checks run on the server. The other three design docs (Measuring Rates, Catch-up Calculation, Delivery & Summary) all build on the definition of a setup given here.

## Chosen approach

A **setup** is one Idle Controller block, a contiguous region of chunks, and the Idle Storage blocks that hold its output and inputs. The player selects the chunks with the Surveyor tool or on a map in the controller's screen, assigns storage using a tool or the storage blocks, and confirms. Nothing is tracked until the player confirms.

### Components

Block and item names are working names.

| Component            | Kind                                                | Role                                                                                                                                                       |
| -------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Idle Controller      | Craftable block                                     | Anchors one setup. Its position identifies the setup, and its screen is where the setup is edited and confirmed.                                           |
| Idle Storage         | Craftable block, tiered                             | Holds a setup's output or inputs. It works like a container, so hoppers can insert and extract, and the player can open it.                                |
| Surveyor             | Craftable tool                                      | Selects chunks in the world and links storage to a controller.                                                                                             |
| Linkable storage tag | Block tag `#idleminecrafter:linkable_storage`       | Storage from vanilla or other mods that the Surveyor may link as an **input** in place of Idle Storage, including hoppers and other transportation blocks. |
| Storage Chain Graphs | Storage chains of linkable storage and idle storage | produce a furthest down the line storage system for chained chests and other storage blocks                                                                |

### Setups and types

- One controller per setup. A world can hold any number of setups.
- The player picks the type on the confirmation screen, and it sets the region limit and which storage roles apply.

| Type                                          | Default max region | Output storage                                   | Input storage                                    |
| --------------------------------------------- | ------------------ | ------------------------------------------------ | ------------------------------------------------ |
| Mob farm                                      | 16 chunks          | Required (Idle Storage or linked tagged storage) | Optional (Idle Storage or linked tagged storage) |
| Quarry (player-built miner, e.g. world eater) | 64 chunks          | Optional (Idle Storage or linked tagged storage) | None                                             |
| Crafting                                      | 9 chunks           | Required (Idle Storage or linked tagged storage) | Required (Idle Storage or linked tagged storage) |

Each limit can be changed separately in the config.

### Regions

- A region is a set of chunks in one dimension. It must include the controller's chunk.
- Chunks must be contiguous: every chunk shares an edge with another chunk in the region, with no gaps. Touching only at a corner does not count. The check is a flood fill from the controller's chunk that must reach every selected chunk.
- Chunks are exclusive: a chunk belongs to at most one setup. The only exception is a joint chunk (below).
- The region may not exceed its type's size limit.

### Selecting chunks

The Surveyor and the controller's map edit the same selection, and every edit is a request the server validates.

- **Surveyor:** sneak-use it on a controller to bind the tool to that controller. While the Surveyor is held, chunk borders are drawn. Using it on the ground toggles that chunk in the bound controller's selection. Colours show selected chunks, chunks owned by another setup, and chunks that would break contiguity.
- **Controller map:** a top-down chunk grid centred on the controller, which scrolls to reach large quarry regions. Clicking or clicking and dragging a chunk toggles it, using the same colours and rules.

### Storage

**Idle Storage** is one block that can serve as input or output. Its role comes from how it is linked, not from what it is.

- **Role per block:** each Idle Storage in a region is assigned a role for that setup (Output or Input) on the controller's screen. Unassigned storage is ignored.
- **Joint blocks**: an idle storage can serve the primary function of it's chunk's owner, and a joined guest can have a separate operation on the same idle storage block.
- **Tiers:** there are nine, in order: wood, copper, iron, gold, amethyst, diamond, Amethyst Cluster (silk-touched), netherite, and collapsed nether star. The collapsed nether star is a new item the mod adds.
- **Upgrades:** within a tier, applying that tier's material raises capacity step by step from the tier's minimum to its maximum. An upgrade made from the next tier's material moves the block up a tier.
- **Capacity:** the block's capacity is the hard ceiling on offline output (see Catch-up Calculation).
- **Hoppers:** they work as on any container. A player's system can feed an input storage or drain an output storage.
- **Linked input storage:** a crafting setup may use existing storage as input instead of Idle Storage. The block must match `#idleminecrafter:linkable_storage`, which includes vanilla chests, barrels and shulker boxes by default, and which other mods or modpacks can extend. It must sit inside the setup's region, and it is linked with the Surveyor. Linking tagged storage as **output** is deferred to v2.0.
- storage chains are automatically put into a graph datastructure so when production is made, items will be filled in the furthest node first and then filled in back to the source as capacity allows. This skips the delay and logic of tick warping hopper chains and distribution streams
- **Upgrade Modules** For v2.0, I want to add upgradability to storage size, the controller blocks, and one upgrade in particular is auto sorting to different storage slices based on items.

### Chaining setups through joint chunks

One setup's output can feed another setup's input. For example, an iron farm's iron and poppies can feed a crafting setup that makes iron blocks and red dye.

- **Joint chunk:** a chunk shared by exactly one pair of controllers. It must share an edge with both regions, and it only counts toward the host region's size limit. The second joined region is the guest region to handle priority.
- **One per pair:** each pair of controllers can have at most one joint chunk, and a chunk can be joint for only one pair.
- **Contents:** the joint chunk holds the shared Idle Storage. Each shared block is the output of one setup and the input of the other, and its direction is set on the controller's screen.
- **One consumer:** each shared storage block feeds at most one setup. To send output to several consumers, the producer can use several storage blocks.
- **No loops:** a link that would let a setup feed itself, directly or through other setups, is refused. Setups therefore always form a chain or tree from producers to consumers.
- Blocks in a joint chunk other than Idle Storage belong to only the host controller. 

### Confirmation

Placing a controller, selecting chunks and assigning storage change nothing until the player presses **Confirm** on the controller's confirmation screen. The screen shows:

- the name and type
- the chunk count against the type's limit, with a chunk map
- each output and input storage, with its tier and capacity
- any joint chunks and their direction of flow
- warnings, and every validation failure

**Confirm** stays disabled while any check fails. Every later change returns the setup to this screen: region, type, storage, links, or joint chunks.

### Validation

The server runs every check at confirmation, and again whenever a registered setup's blocks change.

1. The controller exists, and its chunk is in the region.
2. The region is contiguous, in one dimension, within its type's limit, and overlaps no other setup (joint chunks excepted).
3. There is at least one Output storage. A crafting setup also has at least one Input storage.
4. Each storage block is inside the region or the setup's joint chunk, and is linked to no other setup in the same role.
5. Each linked storage matches `#idleminecrafter:linkable_storage` and is used as an input.
6. The joint-chunk links form no loop.

If a registered setup fails a check (for example, a storage block is broken), it becomes **Invalid**. It stops producing offline output until the player fixes it and confirms again. The player is alerted to this on screen when logging in and with periodic chat messages.

### Lifecycle

| State | Meaning | Offline output |
| --- | --- | --- |
| Draft | Controller placed; selection and storage being edited | No |
| Registered | Confirmed; the mod observes it while the player is present (see Measuring Rates) | Once rates are measured |
| Invalid | A validation check fails | No |
| Removed | Controller broken; the setup's data is deleted, its chunks freed, and its storage becomes ordinary containers | No |

### Data model

Setups are saved in world data (`data/idleminecrafter_setups.dat`) and never in vanilla block or chunk data.

| Field             | Content                                                                                                                                                              |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`              | UUID                                                                                                                                                                 |
| `name`            | Player-chosen name                                                                                                                                                   |
| `type`            | `MOB_FARM`, `QUARRY` or `CRAFTING`                                                                                                                                   |
| `dimension`       | Dimension key                                                                                                                                                        |
| `controller`      | Block position                                                                                                                                                       |
| `chunks`          | Set of chunk positions                                                                                                                                               |
| `storage`         | List of master block position, role (`OUTPUT` or `INPUT`), and source (`IDLE_STORAGE` or `LINKED`) and the master block has a graph of the connected storage chains. |
| `jointChunks`     | List of chunk position, partner setup `id`, and per-storage direction                                                                                                |
| `state`           | `DRAFT`, `REGISTERED` or `INVALID`                                                                                                                                   |
| `registeredAt`    | Game time and real-world time of the last confirmation                                                                                                               |
| `player owner id` | id or name of the player who owns the idle minecrafter for initiating on sign-on.                                                                                    |
|                   |                                                                                                                                                                      |

At load, the mod builds an index from chunk to setup for each dimension. It uses that index for the exclusivity check and for change detection.

### Commands

These are for testing. They need cheats enabled (permission level 2). Later design docs will add to this list.

| Command | Effect |
| --- | --- |
| `/idle list` | Lists setups with id, name, type, state and chunk count |
| `/idle info <setup>` | Shows a setup's full data, including storage and joint chunks |
| `/idle validate <setup>` | Runs validation and prints each check's result |
| `/idle select <setup> add\|remove <chunkX> <chunkZ>` | Edits a selection without the Surveyor |
| `/idle confirm <setup>` | Confirms without the screen (tests only) |
| `/idle remove <setup>` | Unregisters a setup and leaves its blocks in place |
| `/idle prepare-removal` | Replaces every Idle Storage with vanilla chests that hold its contents, so the mod can be removed without losing items |

## Alternatives considered

| Alternative | Why not chosen |
| --- | --- |
| Surveyor only, or controller map only | Both are kept: the tool suits building in the world, the map suits large regions |
| Tag an existing chest as the whole setup | No region, so there is nothing to validate or detect changes against |
| Region as an arbitrary block box | Chunks match how Minecraft loads the world, and are easier to show and check |
| Output split evenly across vanilla containers | Misrepresents builds that route items unevenly, and capacity is harder to know |
| Separate Intake and Supply blocks | One block with roles is simpler, and it is what makes chaining possible |
| Overlapping regions | Double-counts output and makes change detection ambiguous |
| Shared storage inside the producer's region | Joint chunks keep regions exclusive and make the handoff explicit |
| Loops between setups | Catch-up would have to iterate until it settles, which is slower and harder to prove fair |
| Several consumers per storage block | Offline supply would have to be split, which adds a source of unfairness |
| Tagged storage as output | Output measurement would rely on sampling contents; deferred to v2.0 |
| Mod guesses the setup type | The PID requires the player to choose |

## How it meets the essentials

| Essential | How this design meets it |
| --- | --- |
| No save corruption or item loss; the world loads after removal | Setup data lives in its own world-data file, which vanilla ignores. `/idle prepare-removal` turns mod storage back into vanilla chests before the mod is removed. |
| Offline output ≤ online output | Output can only land in Output storage, whose capacity is a hard ceiling. Rates themselves are covered in Measuring Rates. |
| No duplication exploits | A chunk and a storage role belong to one setup only. Each shared storage block has one consumer, and loops are refused, so no item is counted twice. |
| Works with the computer off | Registration is stored data only; nothing runs in the background. |
| Calculations on the server | Every selection edit, link and confirmation is a request the server validates; the client only draws the overlay and screens. |
| Offline time can be capped or disabled | Out of scope for registration; covered by configuration. |

## Hand-offs to the other design docs

- **Measuring Rates:** hoppers can fill output storage from the player's own system. Measurement must separate what the setup produced from what the player deposited. It must also decide which block changes inside a region reset the measurement.
- **Catch-up Calculation:** process chained setups in order from producers to consumers, and cap each setup at its output storage capacity and its input stock. Offline quarries remove the blocks it would have mined.
- **Chained idleminers:** The production during catchup of one output is available to a receiver during it's catchup. So a farm can produce more than the storage allows if a consumer is using it.
- **Delivery & Summary:** offline output lands in Output storage. The summary should list each setup, including any setup that was Invalid and produced nothing. The summary will also state how much of a resource went from one output into the input of another process.


## Open questions

- [ ] Final block and item names.
- [ ] Offline quarries: are mined blocks removed from the world? (To be settled in Catch-up Calculation.)

## Settled: Crafting recipes and storage tiers

All recipes and storage tier capacities are documented in [crafting-recipes.md](crafting-recipes.md).
