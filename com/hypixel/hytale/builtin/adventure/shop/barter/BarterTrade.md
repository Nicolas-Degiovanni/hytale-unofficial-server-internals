---
description: Architectural reference for BarterTrade
---

# BarterTrade

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Data Model

## Definition
```java
// Signature
public class BarterTrade {
```

## Architecture & Concepts
The BarterTrade class is a passive data structure that represents a single, configurable trade between a player and an entity, such as an NPC. It is not an active system component but rather a data model that defines the "recipe" for a transaction: what items are required (input), what item is given (output), and how many times the trade can be performed (stock).

Its primary role within the engine is to be a deserialization target for game data files. The static **CODEC** field is the most critical part of this class, defining the schema and validation rules for loading trade definitions from external configuration files (e.g., JSON or HOCON). This design decouples game logic from specific trade data, allowing designers to define, create, and balance NPC shops without modifying engine code.

Higher-level systems, such as a ShopManager or an NPC's inventory component, will hold collections of BarterTrade objects to represent their complete offering.

## Lifecycle & Ownership
- **Creation:** BarterTrade instances are not intended to be manually instantiated in code. They are created by the Hytale **Codec** system during the asset loading phase. The engine reads a data file defining a shop's inventory and uses the `BarterTrade.CODEC` to deserialize the data into a graph of BarterTrade objects.

- **Scope:** The lifetime of a BarterTrade instance is bound to its owning container, typically an NPC's inventory or a world-based shop. It is loaded when the container is initialized and persists as long as that container exists in the game world.

- **Destruction:** The object is marked for garbage collection when its owning container is destroyed. This occurs, for example, when an NPC is despawned or a game zone is unloaded from memory.

## Internal State & Concurrency
- **State:** Mutable. The internal fields for input, output, and stock are not final. However, this class is designed to be treated as a **read-only record** after its initial creation by the Codec system. Modifying its state at runtime is a significant anti-pattern that can lead to desynchronization and inconsistent game state.

- **Thread Safety:** **This class is not thread-safe.** It contains no internal locking or synchronization primitives. It is expected to be loaded by an asset or I/O thread and subsequently accessed exclusively from the main game logic thread. Unsynchronized access from multiple threads will result in undefined behavior.

## API Surface
The public API is limited to data accessors. The primary interaction point for the *system* is the static CODEC field, which is not intended for direct use in game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOutput() | BarterItemStack | O(1) | Retrieves the item stack that is given to the player upon a successful trade. |
| getInput() | BarterItemStack[] | O(1) | Retrieves the array of item stacks required from the player to execute the trade. |
| getMaxStock() | int | O(1) | Returns the initial maximum number of times this trade can be performed. |

## Integration Patterns

### Standard Usage
A developer should never create a BarterTrade instance directly. Instead, they should retrieve it from a higher-level system that manages an entity's trading capabilities.

```java
// Correctly retrieve pre-loaded trade data from a shop manager or NPC component
ShopComponent shop = npc.getComponent(ShopComponent.class);
List<BarterTrade> availableTrades = shop.getAvailableTrades();

// Use the data to populate UI or validate a transaction
for (BarterTrade trade : availableTrades) {
    BarterItemStack offer = trade.getOutput();
    BarterItemStack[] requirements = trade.getInput();
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BarterTrade()`. All trade definitions must originate from data files to ensure they are correctly managed by the game's content pipeline. This is a data-driven class.

- **Runtime State Mutation:** Do not modify a BarterTrade object after it has been loaded. Changing the input or output at runtime bypasses the data-driven design and can cause unpredictable behavior.
    ```java
    // DANGEROUS: Modifying a loaded trade definition
    BarterTrade trade = shop.getTrade(0);
    trade.maxStock = 999; // This will cause inconsistent state
    ```

## Data Pipeline
The BarterTrade class is a destination in the asset loading pipeline. It does not process data itself; it *is* the data.

> Flow:
> Game Data File (e.g., `village_blacksmith.json`) → Asset Loading Service → `BarterTrade.CODEC` Deserializer → **BarterTrade Instance** → Populates NPC Shop Component → Read by Game Logic & UI Systems

