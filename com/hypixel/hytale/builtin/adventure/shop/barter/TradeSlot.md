---
description: Architectural reference for TradeSlot
---

# TradeSlot

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Polymorphic Base Class

## Definition
```java
// Signature
public abstract class TradeSlot {
```

## Architecture & Concepts
The TradeSlot class is an abstract base class that defines the contract for a single source of trades within a bartering system. It represents a fundamental component in Hytale's data-driven design for NPC merchants and other trading interfaces.

The primary architectural pattern is polymorphism, managed by a static `CodecMapCodec`. This codec acts as a factory and registry, mapping string identifiers ("Fixed", "Pool") to concrete implementations like FixedTradeSlot and PoolTradeSlot. This design decouples the high-level bartering system from the specific logic of how trades are generated. It allows game designers to define complex merchant inventories in data files (e.g., JSON or NBT) which are then deserialized into a list of appropriate TradeSlot objects at runtime.

This class is the cornerstone of a strategy pattern for trade generation. Each concrete subclass implements a different strategy for resolving a set of potential trades into a final list to be presented to the player.

### Lifecycle & Ownership
- **Creation:** TradeSlot instances are not meant to be instantiated directly in code. They are created exclusively through deserialization by the Hytale codec system, using the static `CODEC` field. This process typically occurs when an entity with a bartering component, such as a merchant NPC, is loaded into the world.
- **Scope:** The lifetime of a TradeSlot object is tied to its owning component, which is usually part of a larger entity. It persists as long as the entity's trade data is held in memory.
- **Destruction:** The object is eligible for garbage collection when the owning entity is unloaded or its bartering component is reset or removed. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** As an abstract class, TradeSlot itself is stateless. However, its concrete implementations (FixedTradeSlot, PoolTradeSlot) are stateful. They contain the definitions of trades or trade pools loaded from game data. This state should be considered immutable after initial deserialization.
- **Thread Safety:** This class is **not thread-safe**. The `resolve` method accepts a Random instance, which is inherently not safe for concurrent use. Calls to `resolve` from multiple threads must be externally synchronized, or each thread must use its own unique Random instance to prevent data races and non-deterministic behavior.

## API Surface
The public API is minimal, focusing entirely on the contract for trade generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| resolve(Random var1) | List<BarterTrade> | Varies | **Core Method.** Resolves the configured trade data into a concrete list of trades. Complexity depends on the subclass (e.g., O(1) for Fixed, O(N) for Pool). |
| getSlotCount() | int | O(1) | Returns the number of UI slots this configuration is expected to occupy. |
| CODEC | CodecMapCodec | N/A | Static factory used by the engine to deserialize TradeSlot objects from data files. |

## Integration Patterns

### Standard Usage
The TradeSlot is used by higher-level systems that manage NPC interactions. The system retrieves the list of TradeSlot objects from the NPC's data component and iterates through them to populate the trading UI.

```java
// Pseudo-code for a merchant interaction system
Random tradeRandom = new Random();
List<BarterTrade> availableTrades = new ArrayList<>();

// npc.getTradeSlots() returns a List<TradeSlot> loaded from data
for (TradeSlot slot : npc.getTradeSlots()) {
    availableTrades.addAll(slot.resolve(tradeRandom));
}

// Populate the UI with availableTrades
ui.displayTrades(availableTrades);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new FixedTradeSlot()` or `new PoolTradeSlot()`. The entire system is designed to be configured via data files. Bypassing the codec breaks this data-driven contract and can lead to unmanageable, hardcoded merchant behavior.
- **State Mutation:** Do not attempt to modify the internal state of a TradeSlot subclass after it has been deserialized. These objects should be treated as read-only configurations.
- **Shared Random Instance:** Do not share a single `java.util.Random` instance across multiple threads to call the `resolve` method. This will cause severe concurrency issues.

## Data Pipeline
The TradeSlot is a key transformation step in the pipeline that turns static game data into dynamic player-facing interactions.

> Flow:
> Game Asset (e.g., `npc_merchant.json`) -> Hytale Codec Engine -> **TradeSlot.CODEC** -> Instantiated `TradeSlot` subclass -> `resolve()` call -> `List<BarterTrade>` -> Barter UI System

