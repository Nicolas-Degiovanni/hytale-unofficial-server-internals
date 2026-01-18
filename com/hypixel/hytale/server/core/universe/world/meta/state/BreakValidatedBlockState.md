---
description: Architectural reference for BreakValidatedBlockState
---

# BreakValidatedBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface BreakValidatedBlockState {
   boolean canDestroy(@Nonnull Ref<EntityStore> var1, @Nonnull ComponentAccessor<EntityStore> var2);
}
```

## Architecture & Concepts
The BreakValidatedBlockState interface defines a server-side contract for validating block destruction events. It serves as a critical gate within the world simulation, allowing for complex, state-dependent rules to determine if a block can be broken.

This interface is a core component of the **Strategy Pattern** as applied to block behaviors. Instead of embedding complex conditional logic within the core world engine, the engine delegates the validation decision to a specific implementation of this interface. Each block type that requires custom break validation (e.g., a quest-locked door, an unbreakable dungeon wall) will be associated with a concrete implementation.

The method signature, which requires an EntityStore and a ComponentAccessor, indicates that validation logic is deeply integrated with the Entity Component System (ECS). This allows for powerful and dynamic rules, such as preventing a block's destruction until a specific boss entity is defeated or a certain world flag component is present.

## Lifecycle & Ownership
As an interface, BreakValidatedBlockState has no lifecycle itself. The following applies to its concrete implementations.

-   **Creation:** Implementations are expected to be stateless singletons. They are typically instantiated once at server startup and registered with a central Block Registry or Metadata Manager, which associates them with specific block types.
-   **Scope:** An implementation's lifetime is tied to the server session. It persists as long as the block definitions are loaded in memory.
-   **Destruction:** Implementations are garbage collected during server shutdown when all references from the Block Registry are cleared.

## Internal State & Concurrency
-   **State:** Implementations of this interface **must be stateless**. They should be treated as pure functions whose return value depends exclusively on the provided EntityStore and ComponentAccessor arguments. Storing mutable instance variables is a severe anti-pattern that will lead to unpredictable behavior in a multi-threaded environment.
-   **Thread Safety:** Implementations are required to be thread-safe. The world simulation engine may invoke the canDestroy method from multiple worker threads. Adherence to the statelessness requirement guarantees thread safety. The provided EntityStore context is responsible for its own internal synchronization.

## API Surface
The public contract consists of a single method for validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canDestroy(Ref, ComponentAccessor) | boolean | Implementation-Dependent | Evaluates if a block can be destroyed based on world state. Returns true if destruction is permitted, false otherwise. |

## Integration Patterns

### Standard Usage
This interface is not intended for direct use by gameplay logic developers. It is invoked by the core world simulation engine when processing a block interaction event. The engine retrieves the appropriate implementation from the target block's metadata and uses it as a validation gate.

A conceptual example of the engine's internal logic:
```java
// Engine-level code (conceptual)
Block targetBlock = world.getBlockAt(position);
BlockMeta meta = targetBlock.getMetadata();

// Retrieve the validation strategy for this block type
BreakValidatedBlockState validator = meta.getBreakValidator();

if (validator != null) {
    // Delegate the validation check
    boolean canBeDestroyed = validator.canDestroy(world.getEntityStoreRef(), world.getComponentAccessor());
    if (!canBeDestroyed) {
        // Halt the block destruction process
        return;
    }
}

// Proceed with block destruction...
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Do not create implementations that contain mutable instance fields. This violates the stateless contract and will cause severe concurrency issues.
-   **World State Mutation:** The canDestroy method is a query, not a command. Do not use it to modify the EntityStore or trigger other world events. Such side effects create non-deterministic behavior and are extremely difficult to debug.
-   **Expensive Operations:** Be cautious of performing computationally expensive entity queries or disk I/O within an implementation. This method may be called frequently and can become a performance bottleneck for the server tick.

## Data Pipeline
BreakValidatedBlockState acts as a conditional gateway in the data flow of a block destruction event.

> Flow:
> Player Action -> Server Event Handler -> World Simulation -> **BreakValidatedBlockState.canDestroy()** -> [Conditional] -> Block State Change & Network Sync

