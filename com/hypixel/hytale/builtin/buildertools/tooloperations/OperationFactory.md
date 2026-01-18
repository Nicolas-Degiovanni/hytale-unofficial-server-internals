---
description: Architectural reference for OperationFactory
---

# OperationFactory

**Package:** com.hypixel.hytale.builtin.buildertools.tooloperations
**Type:** Functional Interface / Factory Pattern

## Definition
```java
// Signature
public interface OperationFactory {
```

## Architecture & Concepts
The OperationFactory interface defines a behavioral contract for creating ToolOperation instances. It embodies the Factory Method design pattern, serving as the primary mechanism for decoupling the core builder tool system from the concrete logic of individual tool actions like fill, replace, or copy.

When a player interacts with a builder tool, the server receives a BuilderToolOnUseInteraction packet. The server-side builder tool manager uses a registry to find the appropriate OperationFactory implementation corresponding to the player's selected tool mode. This factory is then invoked to instantiate a new ToolOperation object, which encapsulates the state and logic required to execute that specific world modification.

This architectural choice is critical for extensibility. It allows developers to introduce new, custom builder tool behaviors without modifying the core engine code. By implementing this interface and registering the factory, a new operation becomes a first-class citizen in the builder tool ecosystem.

## Lifecycle & Ownership
As an interface, OperationFactory itself is not instantiated. The lifecycle described here applies to its concrete implementations.

- **Creation:** Implementations are instantiated once during server initialization or plugin loading. They are subsequently registered with a central manager, such as a ToolOperationRegistry.
- **Scope:** A factory implementation is a long-lived, stateless service object. It persists for the entire server session or for the lifetime of the plugin that registered it.
- **Destruction:** The factory object is garbage collected when the server shuts down or its associated plugin is unloaded, which clears the central registry.

## Internal State & Concurrency
- **State:** The OperationFactory contract assumes implementations are **stateless**. They should not contain any mutable fields that persist between calls to the create method. All necessary context for an operation is provided via the method arguments.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. The create method can be invoked concurrently from multiple worker threads, each handling a different player's actions. A stateless implementation is inherently thread-safe.

**WARNING:** Implementing a stateful OperationFactory is a severe anti-pattern and will lead to race conditions and unpredictable world corruption when multiple players use builder tools simultaneously.

## API Surface
The interface exposes a single method, which is its entire public contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create(Ref, Player, BuilderToolOnUseInteraction, ComponentAccessor) | ToolOperation | O(1) | Instantiates a new ToolOperation based on the provided player interaction and world context. |

## Integration Patterns

### Standard Usage
A developer defines a new builder tool behavior by implementing this interface. The factory is then registered with the engine, typically through a dedicated registry service during the server's startup phase.

```java
// 1. Implement the factory for a "Fill" operation
public class FillOperationFactory implements OperationFactory {
    @Nonnull
    @Override
    public ToolOperation create(
        @Nonnull Ref<EntityStore> worldRef,
        @Nonnull Player player,
        @Nonnull BuilderToolOnUseInteraction interaction,
        @Nonnull ComponentAccessor<EntityStore> accessor
    ) {
        // Create a new, unique instance for this specific interaction
        return new FillOperation(worldRef, player, interaction);
    }
}

// 2. Register the factory during server initialization
ToolOperationRegistry registry = context.getService(ToolOperationRegistry.class);
registry.register("hytale:fill", new FillOperationFactory());
```

### Anti-Patterns (Do NOT do this)
- **Stateful Factories:** Do not store any per-player or per-operation state as fields within your factory implementation. This will break concurrency.
- **Returning Shared Instances:** The create method **must** return a new ToolOperation instance on every invocation. Reusing or caching instances will cause catastrophic state corruption, as multiple distinct player actions would be handled by the same object.
- **Direct Invocation:** Developers should never call an OperationFactory directly. Always interact with the high-level builder tool system, which manages the factory lookup and invocation lifecycle.

## Data Pipeline
The OperationFactory acts as a constructor node in the data flow for handling a builder tool command from a client.

> Flow:
> Client Input -> BuilderToolOnUseInteraction (Network Packet) -> Server-Side Tool Manager -> **OperationFactory.create()** -> ToolOperation Instance -> ToolOperation.execute() -> World State Mutation (EntityStore)

