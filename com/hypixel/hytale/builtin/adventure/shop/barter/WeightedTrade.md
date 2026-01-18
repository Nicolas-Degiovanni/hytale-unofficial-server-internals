---
description: Architectural reference for WeightedTrade
---

# WeightedTrade

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Data Model / Configuration Object

## Definition
```java
// Signature
public class WeightedTrade implements IWeightedElement {
```

## Architecture & Concepts
The WeightedTrade class is a data-centric blueprint that defines the static properties of a potential trade within the game's barter system. It is not a live, stateful trade itself, but rather a template from which a live trade can be instantiated. Its primary role is to serve as an in-memory representation of trade data loaded from configuration files, such as JSON or HOCON assets that define an NPC's shop inventory.

As an implementation of the IWeightedElement interface, WeightedTrade objects are designed to be stored in weighted collections. This allows game systems to perform probabilistic selections, where trades with a higher *weight* value are more likely to be chosen when generating a dynamic list of available trades for a character or location.

The key architectural pattern here is the separation of configuration from runtime state. A WeightedTrade instance holds the immutable definition of a trade (inputs, output, potential stock). The factory method *toBarterTrade* is the critical transition point, converting this static template into a stateful BarterTrade object that can be manipulated during gameplay (e.g., its stock can be depleted).

## Lifecycle & Ownership
- **Creation:** WeightedTrade instances are overwhelmingly created via deserialization. The static *CODEC* field is used by the engine's asset loading pipeline to parse game data files and construct these objects. Direct programmatic instantiation via constructors is possible but typically reserved for unit testing or procedural content generation.
- **Scope:** The object's lifetime is tied to the configuration it represents. It persists as long as the owning configuration (e.g., a ShopProfile or an NPC's trade list) is held in memory. It should be treated as a read-only object after its initial creation.
- **Destruction:** Instances are managed by the Java garbage collector. They are eligible for cleanup when their owning configuration is unloaded, such as during a server shutdown or a hot-reload of game assets. There is no explicit destruction logic.

## Internal State & Concurrency
- **State:** The object is **effectively immutable**. Its fields (weight, input, output, stockRange) are populated at creation time by the codec or constructor and are not intended to be modified thereafter. All methods are pure functions that operate on this static data.
- **Thread Safety:** **Thread-safe for read operations.** Because the internal state is constant after initialization, multiple threads can safely invoke methods like getOutput or toBarterTrade without external synchronization.

    **Warning:** When calling methods that require a Random instance, such as resolveStock or toBarterTrade, the calling context is responsible for ensuring the provided Random instance is used in a thread-safe manner.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWeight() | double | O(1) | Retrieves the weight value used by probabilistic selection algorithms. |
| getOutput() | BarterItemStack | O(1) | Returns the immutable definition for the trade's output item stack. |
| getInput() | BarterItemStack[] | O(1) | Returns the immutable array of definitions for the required input item stacks. |
| resolveStock(Random) | int | O(1) | Calculates a concrete stock count from the defined stockRange. |
| toBarterTrade(Random) | BarterTrade | O(1) | **Primary Factory Method.** Instantiates a new, stateful BarterTrade object from this template. |
| toBarterTrade() | BarterTrade | O(1) | **Deprecated.** Avoid usage; it produces deterministic stock values. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve a WeightedTrade from a configuration source and use it as a factory to create a live BarterTrade instance.

```java
// Assume 'shopProfile' is loaded from game data and contains a list of WeightedTrade
List<WeightedTrade> tradeTemplates = shopProfile.getAvailableTrades();

// A higher-level system would select a template, often via a weighted random choice
WeightedTrade selectedTemplate = weightedRandomSelector.select(tradeTemplates);

// The world or session provides the Random instance for consistent behavior
Random worldRandom = server.getWorld().getRandom();

// Convert the configuration template into a live, stateful trade object
BarterTrade liveTrade = selectedTemplate.toBarterTrade(worldRandom);

// This liveTrade can now be added to a specific NPC's active inventory
npc.getShopInventory().addTrade(liveTrade);
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the fields of a WeightedTrade instance after it has been loaded. It is a configuration blueprint and must be treated as immutable to prevent unpredictable behavior across the system.
- **Misusing Deprecated Methods:** Do not call the parameter-less *toBarterTrade()* method in production code. Its deterministic stock generation bypasses the configured stock range, which is almost always undesirable for dynamic game economies.
- **Caching BarterTrade Instances:** Do not cache the result of *toBarterTrade*. The method is designed to be a factory for creating **new** instances. Each call should produce a unique BarterTrade object with its own randomized stock.

## Data Pipeline
The flow of data begins with a static asset file and ends with a dynamic, stateful object in the game world.

> Flow:
> Game Asset (JSON/HOCON) -> Codec Deserializer -> **WeightedTrade** instance -> toBarterTrade() Factory -> BarterTrade (Live Object) -> NPC Shop Inventory

