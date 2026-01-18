---
description: Architectural reference for FixedTradeSlot
---

# FixedTradeSlot

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Transient Data Model

## Definition
```java
// Signature
public class FixedTradeSlot extends TradeSlot {
```

## Architecture & Concepts
The FixedTradeSlot is a concrete implementation of the abstract TradeSlot class. Its primary role within the game's bartering system is to represent a single, deterministic trade that is always available. Unlike other potential TradeSlot implementations that might offer randomized or conditional trades, the FixedTradeSlot guarantees a consistent output.

The most significant architectural feature is the static **CODEC** field. This indicates that FixedTradeSlot is designed to be part of a data-driven system. Game designers define NPC inventories and trade offers in external configuration files (e.g., JSON). The engine's serialization system uses this codec to parse the data and instantiate FixedTradeSlot objects at runtime. This decouples the game logic from the specific trade data, allowing for easy modification of shop contents without recompiling the game.

This class embodies the simplest case in the Strategy pattern implemented by the TradeSlot hierarchy: a strategy that always returns a known, fixed result.

### Lifecycle & Ownership
- **Creation:** Instances are created through two primary pathways:
    1. **Deserialization (Common):** The Hytale codec engine instantiates the class when loading game assets, such as an NPC's shop definition, using the public static CODEC.
    2. **Programmatic (Rare):** Manually instantiated via `new FixedTradeSlot(trade)` by game logic for dynamically generated or temporary trades.
- **Scope:** The lifetime of a FixedTradeSlot instance is tied to its containing object, typically an NPC's inventory or a shop's configuration. It persists as long as that container is held in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is marked for collection when its parent container is unloaded or destroyed. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The class holds a single reference to a BarterTrade object. While the class itself provides no methods to change this reference after construction, the contained BarterTrade object may be mutable. The state of FixedTradeSlot is therefore considered **effectively immutable** post-construction.
- **Thread Safety:** This class is **not thread-safe**. It is a simple data container and is not designed for concurrent access. All interactions with a FixedTradeSlot instance should be confined to the main game thread or a specific world-processing thread to prevent race conditions.

## API Surface
The public API is minimal, focusing on fulfilling the contract of its parent, TradeSlot.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(Random random) | List<BarterTrade> | O(1) | Returns a singleton list containing the fixed trade. The Random parameter is ignored. |
| getTrade() | BarterTrade | O(1) | Provides direct access to the underlying trade definition. |
| getSlotCount() | int | O(1) | Always returns 1, signifying it represents a single trade slot. |

## Integration Patterns

### Standard Usage
A FixedTradeSlot should be treated as an abstract TradeSlot by consuming systems. A higher-level manager, such as a shop controller, will hold a collection of TradeSlot objects and invoke the resolve method to populate the available trades for a player.

```java
// A shop manager retrieves its list of trade slots
List<TradeSlot> slots = npc.getShopInventory().getTradeSlots();
List<BarterTrade> availableTrades = new ArrayList<>();

// The manager resolves all slots to generate the final trade list
for (TradeSlot slot : slots) {
    // The call to resolve is polymorphic. It works for FixedTradeSlot
    // and any other TradeSlot implementation.
    availableTrades.addAll(slot.resolve(world.getRandom()));
}

// The UI is then populated with availableTrades
```

### Anti-Patterns (Do NOT do this)
- **Type Checking:** Do not write code that checks `if (slot instanceof FixedTradeSlot)`. This violates the principle of polymorphism and makes the system brittle. The entire purpose of the TradeSlot abstraction is to handle different slot types uniformly.
- **Assuming Immutability of Contents:** Do not retrieve the BarterTrade via getTrade and modify its state. While the FixedTradeSlot will not change, modifying the shared BarterTrade object can lead to unpredictable behavior across the system.

## Data Pipeline
The primary flow for a FixedTradeSlot is from a data file on disk to an in-game object that influences game logic.

> Flow:
> Game Asset (JSON) -> Hytale Codec Engine -> **FixedTradeSlot.CODEC** -> In-Memory **FixedTradeSlot** Instance -> ShopManager.resolve() -> List<BarterTrade> -> Barter UI Display

