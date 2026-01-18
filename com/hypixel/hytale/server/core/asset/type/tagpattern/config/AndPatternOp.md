---
description: Architectural reference for AndPatternOp
---

# AndPatternOp

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Transient

## Definition
```java
// Signature
public class AndPatternOp extends MultiplePatternOp {
```

## Architecture & Concepts
The AndPatternOp is a concrete implementation of a composite node within the server's tag pattern matching system. This system is designed as a tree-like data structure, or Abstract Syntax Tree (AST), where each node represents a logical test. The AndPatternOp functions as a logical **AND** gate.

It is a container for one or more child TagPattern objects. When its `test` method is invoked, it evaluates each child pattern in sequence. The operation short-circuits and returns false upon the first child that fails its test. It only returns true if *all* contained child patterns evaluate to true.

This class is fundamental for creating complex, rule-based queries on game entities or assets. For example, it can be used to define a rule that matches an item that has *both* a "weapon" tag AND a "magical" tag. These patterns are typically defined in external configuration files and deserialized into this object structure at runtime.

### Lifecycle & Ownership
- **Creation:** Instances of AndPatternOp are not meant to be created directly in game logic. They are instantiated by the Hytale codec system, specifically via the static `CODEC` field, during the deserialization of asset configuration files.
- **Scope:** The lifetime of an AndPatternOp instance is bound to the lifecycle of the parent configuration object that contains it. It persists in memory as long as the asset or rule set it belongs to is loaded.
- **Destruction:** The object is eligible for garbage collection once its parent configuration is unloaded and no longer referenced. The internal `cachedPacket` uses a SoftReference, allowing the garbage collector to reclaim the memory for the cached packet under memory pressure, even if the AndPatternOp itself is still alive.

## Internal State & Concurrency
- **State:** The primary state is the `patterns` array inherited from the parent MultiplePatternOp, which holds the child patterns to be evaluated. This state is considered immutable after deserialization. It also maintains a `cachedPacket` field, which is a mutable, softly-referenced cache for its network packet representation.
- **Thread Safety:** The `test` method is inherently thread-safe as it only performs read operations on its immutable state. However, the `toPacket` method is **not thread-safe**. It performs a check-then-act on the `cachedPacket` field without any synchronization. Concurrent calls to `toPacket` on the same instance can lead to race conditions, potentially creating multiple packet objects.

**WARNING:** Do not share a single AndPatternOp instance across threads if there is a possibility of calling `toPacket` concurrently. The `test` method is safe for concurrent execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(tags) | boolean | O(N) | Evaluates all child patterns. Returns true if all children pass, false otherwise. N is the number of child patterns. Short-circuits on the first failure. |
| toPacket() | TagPattern | O(N) / O(1) | Converts the object to its network protocol representation. The first call is O(N) to build the packet; subsequent calls are O(1) if the cached packet has not been garbage collected. |

## Integration Patterns

### Standard Usage
The primary interaction with this class is through the `test` method. A higher-level system, such as a loot table processor or a crafting recipe validator, will hold a reference to a root TagPattern and invoke `test` against a set of tags representing a specific game context.

```java
// Assume 'pattern' was loaded from a configuration file and is an AndPatternOp
// The tags map represents the object we are testing, e.g., a player's inventory item
Int2ObjectMap<IntSet> tags = ...; // Tags from a game object

if (pattern.test(tags)) {
    // Logic to execute if the object matches the AND condition
    // e.g., "Allow crafting recipe" or "Apply magical effect"
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new AndPatternOp()`. The object graph is complex and must be constructed via the framework's codec system to ensure correctness.
- **State Mutation:** Do not attempt to modify the internal `patterns` array after the object has been created. This will lead to unpredictable behavior.
- **Concurrent Packet Creation:** Avoid calling `toPacket` from multiple threads on the same shared instance. This can defeat the caching mechanism and cause unnecessary object allocation.

## Data Pipeline
The AndPatternOp is a crucial component in the data pipeline that transforms static configuration into dynamic game logic.

> Flow:
> Asset Configuration File (e.g., JSON) -> Server Bootstrap -> **BuilderCodec Deserializer** -> In-Memory **AndPatternOp** instance -> Game System (e.g., RecipeMatcher) -> `test(tags)` invocation -> Boolean Result -> Game Logic Execution

