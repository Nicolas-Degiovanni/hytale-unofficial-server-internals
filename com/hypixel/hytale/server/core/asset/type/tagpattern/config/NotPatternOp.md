---
description: Architectural reference for NotPatternOp
---

# NotPatternOp

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Transient

## Definition
```java
// Signature
public class NotPatternOp extends TagPattern {
```

## Architecture & Concepts
The NotPatternOp class is a concrete node within the tag pattern matching system, which is built upon the **Composite Design Pattern**. It represents a logical NOT operation, inverting the result of a single, nested child pattern.

This class is not a service but a data-centric object, typically deserialized from an asset configuration file (e.g., JSON). It forms part of a larger expression tree of TagPattern objects, where classes like AndPatternOp or OrPatternOp would act as other branches, and a simple tag check would be a leaf.

The primary function of this system is to provide a flexible, data-driven way to query or filter objects (such as entities, items, or blocks) based on a set of assigned tags. NotPatternOp is crucial for expressing exclusion criteria, such as "select all blocks that are *not* wood".

## Lifecycle & Ownership
- **Creation:** NotPatternOp instances are not created directly via their constructor. They are instantiated and configured exclusively by the Hytale codec system, specifically through the static `CODEC` field of type BuilderCodec. This process typically occurs when the server loads and deserializes configuration assets from disk at startup or during a resource reload.
- **Scope:** The lifetime of a NotPatternOp object is bound to its parent configuration asset. It is a short-lived, immutable data object that persists in memory only as long as the asset that defines it is required by the system.
- **Destruction:** There is no explicit destruction or cleanup method. Instances are managed by the Java Garbage Collector and are eligible for collection once the root asset object is no longer referenced.

## Internal State & Concurrency
- **State:** The object's state is defined by its single child, the `pattern` field. While technically mutable during the codec's build process, it should be considered **effectively immutable** after deserialization is complete. It also maintains a `cachedPacket` field, which is a mutable `SoftReference` used for performance caching. This cache can be invalidated by the Garbage Collector under memory pressure.
- **Thread Safety:** This class is **not thread-safe**. While the `test` method is safe for concurrent access as it only performs reads, the `toPacket` method implements a check-then-act pattern on the `cachedPacket` field. Simultaneous calls from different threads can result in a benign race condition where the packet is generated multiple times unnecessarily. This does not corrupt state but leads to redundant work.

**WARNING:** Do not share a NotPatternOp instance across threads if one of them might call `toPacket`. If the object is part of a shared configuration, `toPacket` should be called once and the result cached externally before concurrent processing begins.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(tags) | boolean | O(N) | Evaluates the nested pattern against the provided tags and returns the logical inverse. Complexity N is determined by the child pattern. |
| toPacket() | com.hypixel.hytale.protocol.TagPattern | O(N) | Converts the object into its network-serializable representation. Uses a soft-referenced cache; complexity is O(1) on a cache hit. |

## Integration Patterns

### Standard Usage
Developers do not interact with this Java class directly. Instead, they define the pattern declaratively in an asset file, which the engine then loads.

A conceptual asset file might look like this:
```json
{
  "type": "Not",
  "pattern": {
    "type": "SingleTag",
    "tag": "hytale:is_flammable"
  }
}
```

The system would then use the deserialized object to perform a check:
```java
// Code within the engine's logic, not typical user code
TagPattern pattern = assetManager.load("my_pattern.json");
boolean result = pattern.test(entity.getTags()); // result is true if entity is NOT flammable
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new NotPatternOp()`. The internal `pattern` field will be null, and any subsequent call to `test` or `toPacket` will result in a NullPointerException. Always rely on the codec system for instantiation.
- **Post-Creation Mutation:** Do not modify the public `pattern` field after the object has been constructed by the codec. This violates the assumption of immutability and can lead to unpredictable behavior in systems that have already referenced the object.

## Data Pipeline

The NotPatternOp serves two primary data flows: evaluation and network serialization.

**Evaluation Flow:**
> Asset File (JSON) -> Engine Deserializer (using `CODEC`) -> **NotPatternOp Instance** -> `test(tags)` Method -> Boolean Result

**Network Serialization Flow:**
> **NotPatternOp Instance** -> `toPacket()` Method -> `protocol.TagPattern` DTO -> Network Protocol Encoder -> TCP Packet

