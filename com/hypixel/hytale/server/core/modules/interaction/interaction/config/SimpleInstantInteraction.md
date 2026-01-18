---
description: Architectural reference for SimpleInstantInteraction
---

# SimpleInstantInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Abstract Component Model

## Definition
```java
// Signature
public abstract class SimpleInstantInteraction extends SimpleInteraction {
```

## Architecture & Concepts
SimpleInstantInteraction is an abstract base class that serves as a template for all game interactions designed to execute their primary logic instantaneously. It is a foundational component within the server's Interaction Module, providing a clear contract for "fire-and-forget" actions like consuming an item, flipping a lever, or casting a no-charge spell.

The core design principle is the separation of configuration from execution. Concrete implementations of this class are not services but rather stateless configuration objects, typically deserialized from asset files. The static CODEC field is a strong indicator of this pattern, allowing game designers to define and modify interactions without recompiling the engine.

The "instant" behavior is enforced by its lifecycle methods, which ensure that the primary logic, defined in the abstract **firstRun** method, is invoked exactly once on the very first server tick after the interaction is initiated. This contrasts with other interaction types that might persist over multiple ticks, such as channeling a spell or crafting an item.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the *new* keyword. They are instantiated by the server's asset loading and codec system during server startup or asset reload. The engine uses the static CODEC field to deserialize interaction definitions from configuration files (e.g., JSON) into concrete SimpleInstantInteraction objects.
-   **Scope:** An instance of a SimpleInstantInteraction subclass represents a *template* or *definition*. A single instance, such as one for "EatApple", is loaded once and persists for the entire server session. It is reused for every "EatApple" interaction performed by any entity.
-   **Destruction:** The object is garbage collected when the server's asset registry is cleared, typically during a full server shutdown.

## Internal State & Concurrency
-   **State:** **Stateless and Immutable.** This class, and any conforming subclass, must be treated as a stateless singleton. It holds no per-interaction state. All runtime data, such as the interacting entity or the target block, is provided via the InteractionContext object passed into its methods.

    **WARNING:** Storing mutable instance variables in a subclass is a critical design flaw. Because the same object instance is used for all concurrent interactions of its type, instance variables would create race conditions and unpredictable behavior.

-   **Thread Safety:** The object itself is inherently thread-safe due to its stateless design. However, its methods are part of the core game loop and are **not** designed for concurrent invocation. All methods, including the developer-implemented **firstRun**, will be executed exclusively on the main server thread. Any logic within **firstRun** that modifies shared game state must follow the engine's established patterns for thread safety.

## API Surface
The public contract for developers is fulfilled by implementing the abstract methods. The `tick` methods are part of the internal framework and should not be called directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| firstRun(type, context, cooldownHandler) | abstract void | Varies | **Core Logic Hook.** Subclasses MUST implement this method to define the server-side effect of the interaction. It is guaranteed to be called exactly once. |
| simulateFirstRun(type, context, cooldownHandler) | void | Varies | **Client Prediction Hook.** Defines the client-side predictive effect. By default, it delegates to `firstRun`. Override for custom client-side visuals or logic that must differ from the server. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to extend this class, implement the abstract `firstRun` method, and register the new type within the game's asset configuration. The engine handles the lifecycle.

```java
// In a new file: com/mygame/interactions/ConsumeFoodInteraction.java

// This class would be referenced by name in a JSON asset file.
public class ConsumeFoodInteraction extends SimpleInstantInteraction {

    // This constructor is required for the CODEC system.
    public ConsumeFoodInteraction() {}

    @Override
    protected void firstRun(@Nonnull InteractionType type, @Nonnull InteractionContext context, @Nonnull CooldownHandler cooldownHandler) {
        Entity instigator = context.getInstigator();
        
        if (instigator instanceof Player) {
            Player player = (Player) instigator;
            player.getHungerComponent().addFoodLevel(5); // Apply effect
            player.getInventory().removeItem(context.getItemStack()); // Consume item
        }
        
        // The interaction automatically ends after this method completes.
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ConsumeFoodInteraction()`. The system is responsible for loading and managing these objects from configuration. Direct instantiation bypasses the asset registry and will not function correctly.
-   **Storing State:** Do not add instance fields to store data related to a single interaction. This will cause severe concurrency issues.

    ```java
    // DO NOT DO THIS
    public class BadInteraction extends SimpleInstantInteraction {
        private Player interactingPlayer; // CRITICAL ERROR: This field is shared across all interactions of this type.

        @Override
        protected void firstRun(...) {
            this.interactingPlayer = (Player) context.getInstigator(); // Race condition
        }
    }
    ```
-   **Blocking Operations:** The `firstRun` method executes on the main server thread. Performing database calls, file I/O, or any long-running computation will freeze the entire game loop.

## Data Pipeline
The class acts as a processor within the server's input handling pipeline. It does not transform data but rather consumes an interaction event and produces a change in world state.

> Flow:
> Player Input -> Network Packet -> Server InteractionModule -> **SimpleInstantInteraction.tick0** -> **SimpleInstantInteraction.firstRun** -> World State Change / Entity Component Update

