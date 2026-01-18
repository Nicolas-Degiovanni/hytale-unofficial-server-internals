---
description: Architectural reference for DefaultFluidTicker
---

# DefaultFluidTicker

**Package:** com.hypixel.hytale.server.core.asset.type.fluid
**Type:** Configurable Singleton / Flyweight

## Definition
```java
// Signature
public class DefaultFluidTicker extends FluidTicker {
```

## Architecture & Concepts
The DefaultFluidTicker is the primary server-side implementation for fluid physics simulation in Hytale. It embodies a data-driven design, where specific fluid behaviors like flow speed, spreading logic, and collision interactions are defined in external asset files rather than being hardcoded in Java.

This class acts as a state machine that is executed by the world's block ticking system. For any given fluid block that is scheduled for an update, the engine invokes the `spread` method on the configured DefaultFluidTicker instance. Its core responsibilities are:

1.  **Gravity Simulation:** Attempting to flow downwards into empty or non-solid space.
2.  **Horizontal Spreading:** If downward flow is blocked, spreading horizontally to adjacent blocks with lower fluid levels.
3.  **Collision Resolution:** Defining outcomes when this fluid interacts with a different fluid type. For example, a "water" fluid ticker might have a collision rule for "lava" that results in creating a "stone" block.
4.  **State Management:** It determines the next state of a fluid block and its neighbors, returning a BlockTickStrategy to inform the engine whether the block should continue ticking, sleep, or wait for adjacent chunks to load.

Architecturally, it is decoupled from the concrete world and chunk data structures via the `FluidTicker.Accessor` interface. This allows the core physics logic to operate on an abstraction of the world, enhancing testability and isolating it from changes in world storage implementation.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly by developers. They are instantiated by the Hytale asset loading system when parsing `Fluid` asset definitions. The static `CODEC` field defines how to deserialize the configuration from a file into a DefaultFluidTicker object. A static `INSTANCE` field exists for fallback or default behavior, but configured fluids will have their own unique instances.
-   **Scope:** An instance is tied to a specific `Fluid` asset. It is loaded on server startup and persists for the entire server session, shared by all instances of that fluid in the world.
-   **Destruction:** The object is garbage collected when the server shuts down and the central asset registries are cleared. Its lifecycle is entirely managed by the asset system.

## Internal State & Concurrency
-   **State:** The class holds both configuration state and transient cached state.
    -   **Configuration State:** Fields like `spreadFluid` and `rawCollisionMap` are populated once during asset deserialization. They should be considered immutable post-creation.
    -   **Cached State:** The `collisionMap` and `spreadFluidId` fields are lazily initialized on first access. They convert string-based asset identifiers from the configuration into high-performance integer IDs used during the simulation tick.

-   **Thread Safety:** This class is **not thread-safe**.
    -   The core `spread` method is designed to be called exclusively from the main world tick thread for a given region. It performs direct, unsynchronized reads and writes to chunk data via its `Accessor`. Concurrent calls would lead to world corruption.
    -   The lazy initialization methods (`getCollisionMap`, `getSpreadFluidId`) lack synchronization. This implies a strict engine-level guarantee that the first access to these methods for any given instance will occur from a single thread before it is used in multi-threaded contexts, or that all access occurs on the single world thread.

    **WARNING:** Never call methods on this class from an asynchronous task or a different thread without explicit, engine-provided synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| spread(...) | BlockTickStrategy | O(1) | The primary entry point for the fluid simulation logic. Calculates and applies fluid flow for a single block. Complexity is constant as it only checks a fixed number of neighbors. |
| isSelfFluid(int, int) | boolean | O(1) | Checks if another fluid ID is considered the same as this fluid, including its spreadable form. |
| getCollisionMap() | Int2ObjectMap | O(N) first call, O(1) after | Lazily resolves string-based fluid names from the asset configuration into an integer-keyed map for fast lookups during collision checks. N is the number of collision rules. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Its behavior is defined in fluid asset files. The game engine's world ticking system is the sole consumer of this class. It retrieves the appropriate ticker instance from a `Fluid` asset and invokes `spread` during a scheduled block update.

An example of the *configuration* that drives this class would be in a JSON or HOCON asset file:

```json
// Example: my_lava.json (Fluid Asset)
{
  "ticker": {
    "type": "DefaultFluidTicker",
    "collisions": {
      "hytale:water": {
        "blockToPlace": "hytale:stone",
        "soundEvent": "hytale:sizzle",
        "placeFluid": false
      }
    }
  }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new DefaultFluidTicker()`. This creates an unconfigured instance that lacks the data-driven behavior defined in assets. The system relies on the `CODEC` to properly construct and configure instances.
-   **External State Modification:** Do not attempt to modify the `rawCollisionMap` or other configuration fields after the asset has been loaded. This state is shared globally and modification would lead to undefined behavior.
-   **Calling `spread` Manually:** The `spread` method should only be invoked by the world's block tick scheduler. Calling it directly bypasses crucial engine logic and will corrupt world state.

## Data Pipeline
The DefaultFluidTicker is a processing node in the world simulation pipeline. It is activated by a scheduled event and its output is a modification of world state.

> Flow:
> World Tick Scheduler -> Identifies Ticking Fluid Block at (X,Y,Z) -> Retrieves `Fluid` Asset -> Retrieves configured **DefaultFluidTicker** -> `spread` method invoked with World `Accessor` -> Reads neighbor block/fluid data -> Applies flow/collision logic -> Writes new block/fluid data via `Accessor` -> Returns `BlockTickStrategy` to Scheduler

