---
description: Architectural reference for OrPatternOp
---

# OrPatternOp

**Package:** com.hypixel.hytale.server.core.asset.type.tagpattern.config
**Type:** Transient

## Definition
```java
// Signature
public class OrPatternOp extends MultiplePatternOp {
```

## Architecture & Concepts
The OrPatternOp is a composite node within the server's Tag Pattern evaluation system. This system provides a declarative way to match game assets or entities based on a set of assigned numerical tags. The OrPatternOp implements a logical **OR** condition.

Architecturally, it forms part of an expression tree. Each node in the tree is a subclass of TagPatternOp, representing a specific logical operation (OR, AND, NOT) or a terminal condition (e.g., HasTag). When an asset's tags are evaluated against the root of this tree, the OrPatternOp checks each of its child patterns sequentially. If any child pattern returns a positive match, the OrPatternOp immediately short-circuits and returns true, signifying a successful match for the entire branch.

This component is fundamental for defining complex rules in asset configurations without writing imperative code. For example, it allows a system to specify that a particular sound should play if a block is *either* a type of wood *or* a type of leaf.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are materialized by the static BuilderCodec during the deserialization of server configuration files (e.g., asset definitions). The codec constructs an entire tree of pattern operations from a declarative format.
- **Scope:** The lifetime of an OrPatternOp instance is bound to the configuration object that owns the pattern tree. It is not a global or session-scoped object. It persists in memory as long as the parent asset or rule is loaded.
- **Destruction:** The object is eligible for garbage collection when its containing configuration is unloaded and no longer referenced.

## Internal State & Concurrency
- **State:** The primary state is the `patterns` array inherited from the parent MultiplePatternOp, which holds the child nodes for evaluation. This state is considered immutable after its initial construction by the codec. A secondary, mutable state exists in the `cachedPacket` field, which uses a SoftReference for caching a network-optimized representation of this object.
- **Thread Safety:** The class is **conditionally thread-safe**.
    - The `test` method is fully thread-safe as it only performs read operations on the effectively immutable `patterns` array.
    - The `toPacket` method is not strictly thread-safe due to a potential benign race condition on the `cachedPacket` field. If multiple threads call this method simultaneously on a cold cache, the packet may be generated multiple times. This is not catastrophic as the resulting packet is identical, but it represents redundant work. No locks are used.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(tags) | boolean | O(N) | Evaluates the OR condition against the provided tags. N is the number of child patterns. Short-circuits on the first positive match. |
| toPacket() | TagPattern | O(N) | Converts the object and its children into a network packet. The result is cached in a SoftReference. Complexity is for the initial call; cached calls are O(1). |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. It is used implicitly by higher-level systems that evaluate tag patterns. The system retrieves a pre-configured pattern tree and invokes its `test` method.

```java
// A hypothetical asset manager evaluating a pattern
// The 'asset.getTagPattern()' would return a tree rooted with an OrPatternOp or similar
Int2ObjectMap<IntSet> entityTags = getTagsForEntity(entity);
TagPatternOp pattern = asset.getTagPattern();

if (pattern.test(entityTags)) {
    // Logic for a successful match...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new OrPatternOp()`. The object graph is complex and must be constructed via the provided `CODEC` during configuration loading to ensure correctness.
- **State Mutation:** Do not attempt to modify the internal `patterns` array after the object has been created. The behavior of the pattern evaluation system relies on the immutability of these trees.

## Data Pipeline
The OrPatternOp acts as a transformation and evaluation component within a larger data flow. It does not source data itself but operates on data passed to it.

> **Evaluation Flow:**
> Asset Configuration File -> Server Bootstrap -> **BuilderCodec** -> In-Memory **OrPatternOp** Tree -> Game Logic calls `test(tags)` -> Boolean Result

> **Serialization Flow:**
> In-Memory **OrPatternOp** Tree -> Game Logic calls `toPacket()` -> `com.hypixel.hytale.protocol.TagPattern` -> Network Encoder -> Client

