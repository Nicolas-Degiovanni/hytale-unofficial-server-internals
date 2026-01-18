---
description: Architectural reference for LaunchProjectileInteraction
---

# LaunchProjectileInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Configuration Object

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class LaunchProjectileInteraction extends SimpleInstantInteraction implements BallisticDataProvider {
```

## Architecture & Concepts

The LaunchProjectileInteraction is a server-side, data-driven class that defines the logic for an instantaneous action that creates and launches a projectile entity. It is a concrete implementation within the server's Interaction System, designed to be configured via Hytale's asset loading pipeline rather than being instantiated directly in code.

Its primary architectural role is to act as a self-contained "verb" in the game's Entity Component System (ECS). When an entity performs an action (e.g., using an item, casting a spell), the system can trigger this interaction, which then executes its logic against the world state.

Key architectural characteristics include:

*   **Data-Driven:** The class is defined by a `CODEC`, allowing game designers to create new projectile-launching behaviors in configuration files by specifying a `ProjectileId`. This decouples game logic from game content.
*   **ECS Integration:** It operates exclusively through a `CommandBuffer`. It does not mutate the world state directly. Instead, it queues operations like entity creation (`commandBuffer.addEntity`) and component modification. This is a fundamental pattern in Hytale's server architecture to ensure deterministic, single-threaded updates to the world state at the end of each tick.
*   **State Segregation:** The class itself is stateless at runtime. All necessary context, such as the acting entity (`attackerRef`), the world, and the command buffer, is passed into the `firstRun` method via the `InteractionContext` parameter. This makes the logic highly predictable and testable.
*   **Deprecation:** The `@Deprecated(forRemoval = true)` annotation is a critical warning. This component is scheduled for removal and should not be used for new development. It likely has been superseded by a more flexible or performant system.

## Lifecycle & Ownership

-   **Creation:** Instances are created by the `BuilderCodec` system during server startup or when game assets are loaded/reloaded. It is deserialized from a configuration file (e.g., an item definition) that specifies this interaction type.
-   **Scope:** The object's lifetime is tied to the asset registry. It persists in memory as part of a larger configuration (like an `Item` asset) for the entire duration the server is running. It is not created per-interaction event; the same instance is reused for every execution of that specific interaction.
-   **Destruction:** The object is eligible for garbage collection when the asset registry is cleared, typically during a server shutdown.

## Internal State & Concurrency

-   **State:** The internal state consists of a single field, `projectileId`, which is a String. This field is set once during deserialization by the `CODEC` and is treated as immutable thereafter. The class holds no mutable state related to a specific interaction event.
-   **Thread Safety:** This class is inherently thread-safe due to its immutable state. However, its methods, particularly `firstRun`, are **not** designed to be called from arbitrary threads. They must be executed on the main server thread that processes the game tick. The reliance on a `CommandBuffer` enforces this pattern; accessing it from another thread would corrupt the world state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getProjectileId() | String | O(1) | Returns the configured asset ID of the projectile to be launched. |
| getBallisticData() | BallisticData | O(1) | Retrieves the projectile's configuration from the asset map. Complexity is effectively constant time. |
| firstRun(...) | void | O(1) | Executes the core interaction logic. Queues the creation of a new projectile entity and updates the durability of the held item. **WARNING:** This is the primary entry point and must only be called by the server's Interaction Module. |
| simulateFirstRun(...) | void | O(1) | An empty method intended for client-side prediction. Its lack of implementation indicates that this specific interaction is not predicted on the client. |

## Integration Patterns

### Standard Usage

A developer or game designer does not interact with this class directly in Java code. Instead, it is configured as part of an item's definition file. The server's interaction system automatically invokes it.

A conceptual example of the configuration that would lead to this class being used:

```json
// Example: my_bow.json (Conceptual Item Asset)
{
  "id": "my_bow",
  "type": "WEAPON",
  "interaction": {
    "type": "LaunchProjectileInteraction",
    "ProjectileId": "basic_arrow"
  }
}
```

When a player uses the "my_bow" item, the server's interaction handler finds the `LaunchProjectileInteraction` configuration and calls its `firstRun` method, passing the current `InteractionContext`.

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new LaunchProjectileInteraction()`. The class is designed to be configured and managed by the asset system. Manual creation will result in an uninitialized and non-functional object.
-   **External Invocation:** Do not call the `firstRun` method from outside the server's core interaction processing loop. Doing so will bypass critical systems like cooldowns, context validation, and proper `CommandBuffer` management, leading to world state corruption, crashes, or unpredictable behavior.
-   **Usage in New Systems:** Given the `@Deprecated` status, do not reference or use this class for any new gameplay features. Investigate the replacement system.

## Data Pipeline

The flow of data and control for a projectile launch is managed by the server's core loop. This class is a single, critical step in that pipeline.

> Flow:
> Player Input (Client) -> Network Packet (Server) -> Interaction System -> Item Lookup -> **LaunchProjectileInteraction.firstRun(context)** -> CommandBuffer.addEntity(projectile) -> End of Tick -> CommandBuffer Execution -> World State Updated

