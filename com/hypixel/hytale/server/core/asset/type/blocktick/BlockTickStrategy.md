---
description: Architectural reference for BlockTickStrategy
---

# BlockTickStrategy

**Package:** com.hypixel.hytale.server.core.asset.type.blocktick
**Type:** Enum

## Definition
```java
// Signature
public enum BlockTickStrategy {
   CONTINUE,
   IGNORED,
   SLEEP,
   WAIT_FOR_ADJACENT_CHUNK_LOAD;
}
```

## Architecture & Concepts
BlockTickStrategy is an enumeration that defines a set of control-flow directives for the server-side block ticking scheduler. It serves as a formal contract, dictating how the scheduler should handle the result of a single block tick operation.

Instead of using primitive types like integers or booleans, this enum provides a type-safe, self-documenting, and constrained set of possible outcomes. This prevents invalid states and clarifies the intent of any system that processes block ticks, such as custom block logic or the core world simulation loop. Its primary role is to decouple the *execution* of a block's logic from the *scheduling* of its next update.

## Lifecycle & Ownership
- **Creation:** As a Java enum, all instances are static constants created by the JVM during class loading. They are instantiated once when the BlockTickStrategy class is first referenced and initialized by the ClassLoader.
- **Scope:** The instances are singletons with a global, application-wide scope. They persist for the entire lifetime of the server process.
- **Destruction:** The instances are garbage collected only when the defining ClassLoader is unloaded, which for core engine classes typically only occurs on JVM shutdown.

## Internal State & Concurrency
- **State:** Enum constants are fundamentally immutable. Their internal state, if any, is fixed at compile time and cannot be altered during runtime.
- **Thread Safety:** This enum is inherently thread-safe. Its instances are static, final, and globally accessible singletons. Multiple threads can safely reference these constants without any need for synchronization or locking mechanisms. This is a critical guarantee for the multithreaded world ticking system.

## API Surface
The API surface consists solely of the predefined enum constants, which represent the complete set of valid strategies.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CONTINUE | BlockTickStrategy | O(1) | Signals the scheduler to proceed with the next block tick in the queue immediately. |
| IGNORED | BlockTickStrategy | O(1) | Signals that the tick was consumed but no action is required. The scheduler should discard this tick and move on. |
| SLEEP | BlockTickStrategy | O(1) | Instructs the scheduler to re-queue the current block for a future tick, effectively putting it to sleep for a defined duration. |
| WAIT_FOR_ADJACENT_CHUNK_LOAD | BlockTickStrategy | O(1) | A specialized directive indicating the block tick cannot complete until a neighboring chunk is loaded. The scheduler must suspend this tick and place it in a waiting state, to be re-evaluated upon a chunk load event. |

## Integration Patterns

### Standard Usage
The primary integration pattern is for a block's ticking logic to return one of these enum constants. The calling system, typically the World Ticking Service, then uses a switch statement to execute the appropriate scheduling logic.

```java
// How the world ticking system consumes the strategy
BlockTickStrategy result = block.onTick(world, position);

switch (result) {
    case CONTINUE:
        // Continue to the next block in the queue
        break;
    case SLEEP:
        // Re-schedule this block for a future game tick
        world.getTickScheduler().schedule(position, block, TICK_DELAY);
        break;
    case WAIT_FOR_ADJACENT_CHUNK_LOAD:
        // Add to a list of ticks pending chunk loads
        world.getPendingTickManager().add(position, block);
        break;
    case IGNORED:
        // Do nothing
        break;
}
```

### Anti-Patterns (Do NOT do this)
- **Ordinal Comparison:** Do not rely on the `ordinal()` method for control flow (e.g., `if (strategy.ordinal() == 0)`). This is extremely brittle and will break if the enum order is ever changed. Always compare instances directly or use a switch statement.
- **Extensibility:** Enums are final by design. Do not attempt to extend this set with subclasses; if new strategies are needed, they must be added directly to the enum definition. This ensures a closed, predictable set of outcomes.

## Data Pipeline
This enum acts as a control signal, not a data container. Its value directs the flow of execution within the block update system.

> Flow:
> Block Ticking Logic -> **BlockTickStrategy (Return Value)** -> World Tick Scheduler -> (Decision: Re-queue / Discard / Defer)

