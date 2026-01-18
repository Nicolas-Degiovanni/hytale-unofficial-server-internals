---
description: Architectural reference for BlockTypeUtils
---

# BlockTypeUtils

**Package:** com.hypixel.hytale.builtin.portals.utils
**Type:** Utility

## Definition
```java
// Signature
public final class BlockTypeUtils {
```

## Architecture & Concepts
BlockTypeUtils is a stateless, static utility class designed to centralize the logic for resolving block state variants. In Hytale's asset system, a single conceptual block (e.g., a piston) is represented by multiple BlockType assets, one for each possible state (e.g., extended, retracted). This class provides a canonical, decoupled mechanism for querying these variants.

Its primary architectural function is to act as a stateless transformer. It takes a BlockType and a state identifier string as input and, by querying the central BlockType asset registry, returns the corresponding BlockType asset for that specific state. This prevents systems like world generation, block update logic, or portal mechanics from needing to know the internal details of how block states are defined and linked in the asset files.

This utility is fundamental for any code that needs to dynamically change a block from one state to another without hardcoding asset keys.

### Lifecycle & Ownership
- **Creation:** As a static utility class with a private constructor, BlockTypeUtils is never instantiated. The Java Virtual Machine loads the class definition into memory when it is first referenced.
- **Scope:** The class and its static methods are available globally for the entire application lifetime, bound to the lifecycle of its ClassLoader.
- **Destruction:** There are no instances to manage or destroy. The class definition is unloaded by the JVM during application shutdown.

## Internal State & Concurrency
- **State:** BlockTypeUtils is completely stateless. It contains no member variables and does not cache any data. Each method call is an independent, idempotent operation.
- **Thread Safety:** This class is inherently thread-safe. Its methods are static, operate only on their given parameters, and hold no internal state. Multiple threads can safely call its methods concurrently.
    - **Warning:** The thread safety of this class is dependent on the thread safety of the global asset registry it queries, specifically `BlockType.getAssetMap()`. Concurrent modification of the asset map while this utility is being used could lead to unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockForState(BlockType, String) | BlockType | O(1) | Resolves a specific block state variant. First, it determines the base BlockType and then queries that base for the requested state. Returns null if the state does not exist. |

## Integration Patterns

### Standard Usage
This utility should be used whenever a system needs to find a specific variant of a block based on a state name. The caller provides any BlockType belonging to the family and the desired state.

```java
// Assume 'pistonBlock' is the BlockType for a retracted piston.
// We need to find the BlockType for the 'extended' state.

BlockType pistonBlock = assetManager.get("hytale:piston");
BlockType extendedPiston = BlockTypeUtils.getBlockForState(pistonBlock, "extended");

if (extendedPiston != null) {
    world.setBlock(position, extendedPiston);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Attempting to create an instance via `new BlockTypeUtils()` will result in a compile-time error due to the private constructor. This class is not designed to be instantiated.
- **Assuming Non-Null Return:** The `getBlockForState` method may return null if the requested state does not exist or if the asset registry is malformed. Always perform a null check on the result before use to prevent `NullPointerException`.
- **Passing Null Arguments:** Providing a null `blockType` or `state` argument will result in a `NullPointerException` within the method. Input validation must be performed by the caller.

## Data Pipeline
The data flow for this component is a simple, synchronous transformation. It does not participate in any asynchronous event loops or complex data streams.

> Flow:
> Input `BlockType` -> **BlockTypeUtils.getBlockForState** -> Queries `BlockType.getAssetMap()` -> Returns resolved `BlockType` Variant

