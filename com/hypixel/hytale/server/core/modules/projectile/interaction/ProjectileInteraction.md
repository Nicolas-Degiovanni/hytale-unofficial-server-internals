---
description: Architectural reference for ProjectileInteraction
---

# ProjectileInteraction

**Package:** com.hypixel.hytale.server.core.modules.projectile.interaction
**Type:** Transient

## Definition
```java
// Signature
public class ProjectileInteraction extends SimpleInstantInteraction implements BallisticDataProvider {
```

## Architecture & Concepts

The ProjectileInteraction class is a concrete implementation of the server-side `Interaction` system. Its primary architectural role is to serve as a bridge between a generic player action (an "interaction") and the specialized `ProjectileModule`. It represents the act of firing a projectile, such as an arrow from a bow or a magic spell.

This class follows a data-driven design pattern. An instance of ProjectileInteraction does not contain the specific physics or visual data of a projectile. Instead, it holds a single string reference, `config`, which is an asset ID for a `ProjectileConfig` file. This decouples the interaction *logic* from the projectile *data*, allowing designers to create new projectile types by simply writing new configuration files without modifying engine code.

A critical aspect of its design is the client-authoritative initiation. The `getWaitForDataFrom` method returns `WaitForDataFrom.Client`, signaling to the interaction system that the server must wait for precise position and rotation data from the client before executing the interaction. This is a deliberate design choice to minimize latency perception for the player; the projectile originates exactly where the player was aiming on their screen at the moment of firing. The server then takes this client data as the ground truth for spawning the projectile in the world simulation.

## Lifecycle & Ownership

-   **Creation:** Instances of ProjectileInteraction are not instantiated directly using the `new` keyword. They are deserialized from game asset files (e.g., JSON) by the engine's asset loading system, which uses the static `CODEC` field. Each instance represents a single, reusable interaction definition.
-   **Scope:** An instance is loaded once when the server boots or reloads assets. It is then held in a central registry managed by the `InteractionModule`. The same instance is reused for every execution of that specific interaction type throughout the server's lifetime.
-   **Destruction:** The object is eligible for garbage collection only when the server unloads its asset registries, typically during a full shutdown or a hot-reload of game data.

## Internal State & Concurrency

-   **State:** The class is effectively immutable after its initial creation from configuration. Its only significant state is the `config` string, which is an asset identifier. All other data required for execution is passed into its methods via the `InteractionContext` parameter, making the class itself stateless with respect to any single interaction event.
-   **Thread Safety:** This class is thread-safe. Because its internal state is immutable post-deserialization, a single cached instance can be safely used to process thousands of concurrent interaction events from different players across multiple threads. All mutable state is contained within the `InteractionContext` and `CommandBuffer` objects, which are assumed to be thread-local or otherwise managed by the calling Entity Component System (ECS).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getConfig() | ProjectileConfig | O(1) | Resolves the string ID into a cached `ProjectileConfig` asset. Returns null if the asset is not found. |
| getBallisticData() | BallisticData | O(1) | Provides the projectile's physical properties. Delegates to `getConfig()`. |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | **CRITICAL:** Specifies that the server must wait for client aim data before execution. |
| firstRun(...) | void | O(1) | The primary execution logic. Spawns the projectile using data from the `InteractionContext`. |
| simulateFirstRun(...) | void | O(1) | Populates the `InteractionSyncData` with the server's view of the attacker's state for client-side prediction. |

## Integration Patterns

### Standard Usage

A developer does not typically invoke this class's methods directly. The engine's `InteractionModule` triggers it based on player actions. The primary integration is through data configuration, where an item or skill is defined to use this interaction.

The following example shows the conceptual flow within the interaction system when this class is invoked.

```java
// This code is executed by the server's InteractionModule, not by end-users.
// 'context' is pre-populated with player and world data.
// 'interaction' is the cached instance of ProjectileInteraction.

// The server has received a packet from the client with aim data.
interaction.firstRun(InteractionType.PRIMARY, context, cooldownHandler);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new ProjectileInteraction()`. The object will be unconfigured and will cause `NullPointerException` or `IllegalStateException` when used. It must be loaded from asset files.
-   **Server-Side State Guessing:** Do not attempt to bypass the client-provided state in `firstRun`. The `hasClientState` check is essential. Spawning a projectile based only on the server's last known entity transform will result in severe desynchronization between the client and server.
-   **Invalid Configuration:** Ensure that the `config` ID provided in the asset files points to a valid and loaded `ProjectileConfig`. If `getConfig()` returns null, the interaction will fail silently. The `configurePacket` method includes a runtime check that will throw an `IllegalStateException` if the config is invalid when the interaction is networked.

## Data Pipeline

The flow of data for a single projectile interaction event is client-initiated and server-validated.

> Flow:
> Client Input -> Client calculates aim vectors -> Client sends `Interaction` packet with `InteractionSyncData` -> Server Network Layer -> Server `InteractionModule` -> **`ProjectileInteraction.firstRun`** -> `ProjectileModule.spawnProjectile` -> `CommandBuffer` appends "Create Entity" command -> ECS processes command buffer, spawning the projectile entity in the world.

