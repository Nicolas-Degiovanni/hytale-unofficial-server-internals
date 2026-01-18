---
description: Architectural reference for SimpleBlockInteraction
---

# SimpleBlockInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.client
**Type:** Abstract Component

## Definition
```java
// Signature
public abstract class SimpleBlockInteraction extends SimpleInteraction {
```

## Architecture & Concepts

The SimpleBlockInteraction class is an abstract, server-side component that serves as the foundation for all player interactions targeting a specific block in the world. It is a critical part of the server's configuration-driven Interaction System, designed to be extended by concrete interaction types such as placing a block, tilling soil, or opening a container.

This class establishes a formal contract for a **client-initiated, server-validated** interaction model. The client predicts the interaction for responsiveness, but the server holds ultimate authority over its execution. This is achieved through a dual-tick system:

*   **tick0:** The authoritative server-side execution logic. It performs validation checks (e.g., reach distance) and executes the core game logic.
*   **simulateTick0:** The predictive client-side simulation. It provides immediate feedback to the player while the server processes the authoritative request.

A key architectural feature is the `useLatestTarget` flag. When true, the interaction dynamically uses the block the player is currently looking at, as reported by the client. When false, it uses the block that was targeted when the interaction was first initiated. This allows for two distinct behaviors: "fire-and-forget" interactions and "channeled" or "held" interactions that continuously update their target.

Instances of this class are not created programmatically. Instead, they are deserialized from asset configuration files via the static `CODEC` field, allowing game designers to define and modify block interactions without changing engine code.

### Lifecycle & Ownership

-   **Creation:** Instances are deserialized from configuration assets at server startup by the engine's Codec system. The static `CODEC` field is the exclusive entry point for instantiation. Direct construction is an anti-pattern.
-   **Scope:** An instance of a SimpleBlockInteraction subclass is a stateless template. It is loaded once and persists for the entire lifetime of the server process, shared across all players and interactions of its type.
-   **Destruction:** The object is garbage collected during server shutdown when the central interaction and asset registries are cleared.

## Internal State & Concurrency

-   **State:** This class is designed to be **stateless and immutable** after its initial creation from configuration. The `useLatestTarget` field is set during deserialization and is not modified at runtime. All transient, per-interaction state is stored within the `InteractionContext` object passed into its methods.

-   **Thread Safety:** The object instance itself is thread-safe due to its immutable nature. However, its methods are **not re-entrant** and are designed to be invoked exclusively by the server's main game loop thread. The `InteractionContext` and `CommandBuffer` parameters are not thread-safe and must not be accessed from other threads.

    **Warning:** Invoking `tick0` or `simulateTick0` from outside the main engine tick loop will lead to race conditions, world corruption, and server instability.

## API Surface

The primary API surface consists of abstract methods that concrete subclasses must implement to define specific game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick0(...) | protected void | O(N) | Server-side entry point. Validates and executes the interaction. Calls `interactWithBlock`. |
| simulateTick0(...) | protected void | O(N) | Client-side entry point. Predicts the interaction outcome. Calls `simulateInteractWithBlock`. |
| interactWithBlock(...) | protected abstract void | Varies | **Implementation Required.** Defines the authoritative server logic for the block interaction. |
| simulateInteractWithBlock(...) | protected abstract void | Varies | **Implementation Required.** Defines the predictive client-side logic for the interaction. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `WaitForDataFrom.Client`, indicating the server waits for client sync data before running `tick0`. |

## Integration Patterns

### Standard Usage

Developers do not instantiate or call this class directly. The standard pattern is to extend it, implement the abstract methods, and register the new interaction type in the game's asset configuration.

```java
// Example: A concrete implementation for placing a torch.
// This class would be defined and then referenced in a JSON asset file.

public class PlaceTorchInteraction extends SimpleBlockInteraction {

    // The engine will deserialize this object using its CODEC.
    public PlaceTorchInteraction() {
        super("hytale:place_torch");
    }

    @Override
    protected void interactWithBlock(
        @Nonnull World world,
        @Nonnull CommandBuffer<EntityStore> commands,
        @Nonnull InteractionType type,
        @Nonnull InteractionContext context,
        @Nullable ItemStack itemInHand,
        @Nonnull Vector3i blockPos,
        @Nonnull CooldownHandler cooldownHandler
    ) {
        // Authoritative server logic:
        // 1. Check if the itemInHand is a torch.
        // 2. Check if the target block face is valid for placing a torch.
        // 3. Consume the item from the player's inventory.
        // 4. Use the CommandBuffer to place a torch block in the world.
        // 5. Set interaction state to Succeeded.
    }

    @Override
    protected void simulateInteractWithBlock(
        @Nonnull InteractionType type,
        @Nonnull InteractionContext context,
        @Nullable ItemStack itemInHand,
        @Nonnull World world,
        @Nonnull Vector3i blockPos
    ) {
        // Predictive client logic:
        // 1. Visually place a "ghost" torch block for immediate feedback.
        // This will be corrected by the server's authoritative state later.
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new PlaceTorchInteraction()`. The interaction system relies on the `CODEC` to load and register interactions from assets. Manual instantiation will result in an unregistered component that the engine cannot execute.
-   **Stateful Subclasses:** Do not add mutable fields to subclasses. All state related to a *single execution* of an interaction must be stored in the `InteractionContext` or its `MetaStore`. Storing state on the interaction object itself will cause data corruption, as the same instance is shared by all players.
-   **Omitting Super Calls:** In `tick0` or `simulateTick0` overrides, failing to call `super.tick0(...)` will break the interaction's lifecycle management, preventing it from correctly transitioning to `Failed` or `Succeeded` states.

## Data Pipeline

The execution of a SimpleBlockInteraction follows a well-defined data flow from client input to server-side world modification.

> Flow:
> Client Input (e.g., Mouse Click) -> Client predicts interaction via **simulateTick0** -> Client sends `StartInteraction` network packet with target data -> Server receives packet -> Interaction Module resolves to the correct **SimpleBlockInteraction** instance -> Server invokes **tick0** -> Validation (distance, permissions) -> **interactWithBlock** is executed -> World state is modified via `CommandBuffer` -> State changes are replicated to all clients.

