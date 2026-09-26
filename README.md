# RemsTales

Windower 4 addon that coordinates Rem's Tale lotting across local FFXI clients.

When Rem's Tale chapters enter a shared treasure pool, every loaded RemsTales client reports its current chapter totals. One deterministic coordinator then assigns each page to the eligible character with the lowest total for that chapter.

The effective total is:

```
physical copies in local bags + copies stored with Monisette
```

The addon does **not** auto-pass Rem's Tale pages on the other clients.

## What it tracks

RemsTales recognizes the ten Rem's Tale items:

- Ch.1 = item 4064
- Ch.2 = item 4065
- ...
- Ch.10 = item 4073

Physical copies are counted in:

- Inventory
- Mog Safe
- Mog Safe 2
- Storage
- Mog Locker
- Mog Satchel
- Mog Sack
- Mog Case

Monisette storage is read from the Currencies I packet (`0x113`). The addon requests that data with the normal Currencies I request (`0x10F`).

A client with no fresh Monisette snapshot is excluded from automatic allocation rather than being treated as if it owns zero stored pages.

## Coordination model

Every client that sees the same treasure pool participates through Windower IPC.

The pool identity includes:

- zone
- treasure slot
- item id
- server treasure timestamp

That means unrelated local Windower instances do not participate just because the addon is loaded.

For each Rem's Tale pool:

1. RemsTales waits briefly for the drop burst to settle.
2. Each matching local client rescans all physical Rem's Tale copies.
3. Each client reports its Monisette totals and Inventory capacity.
4. A deterministic local coordinator is elected.
5. Pages are allocated in treasure-slot order.
6. The selected client lots the assigned slot.
7. Server lot activity is monitored and failed assignments can fall back to another eligible client.

## Intelligent balancing

Allocation uses a virtual balance while processing the current pool.

Example:

```
Etamame   Ch.5 = 10
Nyoourke  Ch.5 = 11
Terrasjr  Ch.5 = 20

Three Ch.5 pages drop.
```

The allocator does not blindly give all three pages to Etamame. Each assignment virtually increments the winner before the next page is evaluated, so the pages are distributed toward the lowest running totals.

Equal totals use a deterministic tie-breaker.

## Inventory safety

A character is eligible for a chapter only when the page can actually land in main Inventory.

RemsTales accounts for both:

- an existing non-full stack of that chapter in Inventory
- free Inventory slots

Capacity is reserved while multiple pages in the same pool are allocated, so a character with one free slot will not be assigned several different chapter stacks that cannot all fit.

## Commands

```text
//rt status
//rt totals
//rt peers
//rt on
//rt off
//rt observe
//rt observe on
//rt observe off
//rt refresh
//rt verbose
//rt verbose on
//rt verbose off
```

### `//rt status`

Shows local mode, Monisette snapshot freshness, current pool state, peer count, and coordinator.

### `//rt totals`

Shows this character's physical, stored, and combined counts for chapters 1-10.

### `//rt peers`

Shows the peer totals collected for the current/most recent pool cycle.

### `//rt on` / `//rt off`

Enables or disables this client as an allocation participant.

The default is **enabled**.

### `//rt observe [on|off]`

Observe mode participates in coordination and computes assignments but does not lot.

If every participating client is in observe mode, the entire pool becomes a dry run. This is useful for validating counts and assignment behavior before enabling automatic lotting.

If any valid auto-lot clients are present, observe-only clients are not selected as winners.

### `//rt refresh`

Requests a fresh Monisette/Currencies I snapshot.

### `//rt verbose [on|off]`

Enables additional coordination diagnostics.

## Installation

Place the repository folder at:

```text
Windower4/addons/RemsTales/
```

Load it on every local character that should participate:

```text
//lua l RemsTales
```

For automatic startup, add the load command to the appropriate Windower init configuration.

## Recommended first test

Load RemsTales on every participating client and put all of them into observe mode:

```text
//rt observe on
```

Then run an HTMB that drops Rem's Tales and confirm that the logged assignment matches:

```text
//rt totals
//rt peers
```

Once the totals and assignments look correct:

```text
//rt observe off
```

## Coexistence with Treasury / other auto-lot addons

Do not configure another addon to automatically lot the same Rem's Tale pages.

RemsTales intentionally leaves non-winning clients alone instead of auto-passing. Another addon that independently lots Rem's Tales can defeat the single-winner coordination model.

## Version 0.1.0

- Coordinate local Windower clients through IPC.
- Count physical chapters across normal storage bags.
- Read Monisette chapter balances from Currencies I.
- Identify shared treasure pools using slot/item/timestamp signatures.
- Allocate each chapter to the lowest-total eligible client.
- Virtually rebalance repeated copies of the same chapter in one pool.
- Track Inventory stack/free-slot capacity across assignments.
- Support deterministic tie breaking.
- Support observe-only dry runs.
- Retry local lot requests and monitor server lot activity.
- Avoid auto-passing non-winning clients.
