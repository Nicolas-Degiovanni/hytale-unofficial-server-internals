---
description: Architectural reference for EqualsTagOp
---

# EqualsTagOp

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Transient

## Definition
```java
// Signature
public class EqualsTagOp extends TagPattern {
```

## Architecture & Concepts
The EqualsTagOp is a concrete implementation of the TagPattern strategy. It represents the most fundamental conditional check within the server's tag-based rule systems: "Does a given entity or object possess a specific tag?".

This class is not a service but a data-driven configuration object. It is designed to be defined in external asset files (e.g., JSON) and loaded at runtime by the server's asset management system. Its primary role is to act as a predicate in a larger evaluation tree, enabling complex logic for systems like loot tables, mob spawning rules, or quest triggers.

A critical design aspect is the translation of a human-readable string tag into a high-performance integer index. During the asset loading phase, the string `tag` is resolved against the global AssetRegistry to get an integer `tagIndex`. All subsequent `test` operations use this integer index for O(1) hash map lookups, avoiding costly string comparisons during the game loop.

## Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale `CODEC` system during server bootstrap or asset reloading. The static `CODEC` field defines the deserialization logic, including the crucial `afterDecode` hook which resolves the tag string into its integer index. Programmatic instantiation via `new EqualsTagOp("tag")` is possible but uncommon.
- **Scope:** The lifetime of an EqualsTagOp instance is bound to its parent configuration object. It is a value object within a larger data structure and does not persist globally or across sessions.
- **Destruction:** The object is eligible for garbage collection as soon as its containing configuration is unloaded or goes out of scope. The `cachedPacket` uses a SoftReference, indicating that its cached network representation can be reclaimed by the garbage collector under memory pressure, even if the EqualsTagOp itself is still referenced.

## Internal State & Concurrency
- **State:** The object's state is mutable upon creation. The `tag` string is set, and the `tagIndex` is subsequently populated by the `afterDecode` lifecycle hook. After this initial hydration phase, the object should be treated as immutable. The `cachedPacket` field is a mutable cache that is populated on-demand.
- **Thread Safety:** **This class is not thread-safe.** Its fields are accessed without any synchronization primitives. The `toPacket` method, which lazily initializes `cachedPacket`, contains a check-then-act race condition. All interactions with an EqualsTagOp instance must be confined to a single thread, typically the main server thread or a dedicated asset loading thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Int2ObjectMap tags) | boolean | O(1) | Evaluates the predicate. Returns true if the provided tag map contains the integer index associated with this operation. |
| toPacket() | TagPattern | O(1) | Converts the server-side configuration object into a lightweight, network-transferable representation for the client. The result is cached in a SoftReference for subsequent calls. |

## Integration Patterns

### Standard Usage
This class is not meant to be used in isolation. It is evaluated by a higher-level system that provides the tag context.

```java
// Assume 'rule' is a configuration object holding a TagPattern
// Assume 'entityTags' is the set of tags on a game entity

// This EqualsTagOp is likely part of a larger rule definition
TagPattern condition = rule.getCondition(); // Returns an EqualsTagOp instance

// The game logic evaluates the condition against an entity
boolean result = condition.test(entityTags);

if (result) {
    // Grant loot, trigger behavior, etc.
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the public `tag` field after the object has been initialized by the asset loader. Doing so will cause a desynchronization with the internal `tagIndex`, leading to incorrect behavior in the `test` method.
- **Concurrent Access:** Do not share an instance of EqualsTagOp across multiple threads. Calling `test` or `toPacket` from different threads can lead to unpredictable behavior and race conditions, especially with the lazy initialization of the cached packet.

## Data Pipeline
The primary data flow for this class involves its creation from static assets and its subsequent use in runtime evaluation.

> **Asset Loading Flow:**
> JSON Asset File -> Hytale Codec Deserializer -> **EqualsTagOp** (In-Memory Object with `tagIndex` resolved)

> **Runtime Evaluation Flow:**
> Game Event -> Rule Engine retrieves **EqualsTagOp** -> `test(entityTags)` -> Boolean Result -> Game Logic Branch

> **Network Synchronization Flow:**
> Server-side **EqualsTagOp** -> `toPacket()` -> Protocol TagPattern -> Network Packet -> Client
---

