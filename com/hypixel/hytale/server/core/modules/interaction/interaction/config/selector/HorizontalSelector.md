---
description: Architectural reference for HorizontalSelector
---

# HorizontalSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration & Factory

## Definition
```java
// Signature
public class HorizontalSelector extends SelectorType {
```

## Architecture & Concepts

The HorizontalSelector is a data-driven configuration class that defines a sweeping, arc-shaped volume used for hit detection. It is a fundamental component of the server-side interaction system, designed to model actions like sword swings, cleaves, or other horizontal area-of-effect abilities.

Architecturally, this class embodies the **Flyweight** and **Factory** patterns.
1.  **Configuration (Flyweight):** The primary HorizontalSelector object is an immutable blueprint. It is deserialized from configuration files via its static CODEC field. This allows designers to define complex hit shapes in data files without writing code. A single instance of this configuration can be shared across all game objects that use the same interaction.
2.  **Runtime Instance (Factory):** The class acts as a factory for a private inner class, RuntimeSelector, via the newSelector method. The RuntimeSelector is a short-lived, stateful object that executes the actual hit detection logic for a single interaction event. This pattern cleanly separates the immutable definition of the selector from the mutable state required to process a single, time-based action.

Under the hood, hit detection is not performed with simple geometric checks. Instead, it leverages a sophisticated system based on 3D graphics projection. The RuntimeSelector configures a HitDetectionExecutor with a FrustumProjectionProvider. This allows it to define the sweeping arc as a series of narrow view frustums, providing a highly accurate and configurable 3D volume for collision testing against entity and block bounding boxes.

## Lifecycle & Ownership

-   **Creation:** The HorizontalSelector configuration object is not instantiated directly. It is created by the engine's Codec system during server startup or when asset packs are loaded. The static CODEC field defines how to parse the configuration from a data source.
-   **Scope:** A configured HorizontalSelector instance is a long-lived, shared object. It persists for the entire server session, typically held in a central asset registry. The stateful inner RuntimeSelector, however, has a very short scope. A new instance is created by calling newSelector at the beginning of an interaction and is eligible for garbage collection as soon as the interaction completes.
-   **Destruction:** The HorizontalSelector configuration object is destroyed when the server shuts down or performs a full asset reload. The RuntimeSelector instance is destroyed by the garbage collector after its use.

## Internal State & Concurrency

-   **State:**
    -   **HorizontalSelector (Outer Class):** Effectively **immutable**. Its properties (yawLength, endDistance, etc.) are set once during deserialization and are not modified at runtime. This makes the configuration object inherently safe to share.
    -   **RuntimeSelector (Inner Class):** Highly **mutable**. It maintains state for a single, in-progress interaction, including timing information (lastTime, runTimeDeltaPercentageSum) and the configured HitDetectionExecutor.

-   **Thread Safety:**
    -   **HorizontalSelector:** **Thread-safe**. As an immutable object, it can be safely accessed from any thread.
    -   **RuntimeSelector:** **NOT thread-safe**. This object is designed to be owned and operated by a single thread, typically the main server thread responsible for ticking the entity performing the interaction. Its state is updated sequentially on each call to the tick method. Concurrent access will corrupt its internal timing state and lead to undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Factory method. Creates a new, stateful runtime instance for executing a single interaction. |
| toPacket() | com.hypixel.hytale.protocol.Selector | O(1) | Serializes the selector's configuration into a network packet. Used to inform clients about the interaction's shape, likely for rendering visual effects. |

## Integration Patterns

### Standard Usage

The HorizontalSelector is intended to be retrieved from a registry or asset manager, not created manually. A game system, such as a combat handler, initiates an interaction by creating a new RuntimeSelector and ticking it over the interaction's duration.

```java
// 1. Retrieve the pre-configured selector (e.g., for a "broadsword_swing")
HorizontalSelector swingSelector = AssetManager.get(HorizontalSelector.class, "broadsword_swing");

// 2. When an entity attacks, create a new runtime instance
Selector activeSwing = swingSelector.newSelector();

// 3. For the duration of the attack, tick the selector and query for hits
// This would typically occur inside an entity's update loop
activeSwing.tick(commandBuffer, attackerRef, currentTime, duration);
activeSwing.selectTargetEntities(commandBuffer, attackerRef, (hitEntity, hitLocation) -> {
    // Apply damage or effects to hitEntity
});
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new HorizontalSelector()`. The object is designed to be configured and created via its CODEC. Manual instantiation will result in an unconfigured and non-functional object.
-   **RuntimeSelector Reuse:** Do not cache and reuse a RuntimeSelector instance for multiple interactions. It is stateful and its internal timing calculations are specific to a single event. Always call newSelector to begin a new interaction.
-   **State Modification:** Do not attempt to modify the public fields of the HorizontalSelector configuration object after it has been loaded. While not declared final, they are treated as immutable and changing them at runtime can have unpredictable side effects across the server.

## Data Pipeline

The data flow describes the process within a single tick of the RuntimeSelector instance.

> Flow:
> Attacker Transform & HeadRotation -> **tick()** -> FrustumProjectionProvider Configuration -> HitDetectionExecutor Setup -> **selectTargetEntities() / selectTargetBlocks()** -> Bounding Box Collision Test -> Hit Result (Entity or Block)

1.  **Input:** The `tick` method receives the attacker's current world state (position, orientation) and timing information from the game loop.
2.  **State Update:** It calculates the progress of the swing for the current frame (`runTimeDeltaPercentage`).
3.  **Volume Definition:** Based on this progress, it configures the `FrustumProjectionProvider` to define the precise 3D shape of the swing's active segment for that frame.
4.  **Line of Sight (Optional):** If `testLineOfSight` is true, it configures the `HitDetectionExecutor` with a `LineOfSightProvider` that performs a block-by-block raycast through the world to ensure there are no solid obstacles between the attacker and potential targets.
5.  **Collision Test:** The `selectTargetEntities` and `selectTargetBlocks` methods use the fully configured `HitDetectionExecutor` to perform efficient frustum-vs-AABB (Axis-Aligned Bounding Box) intersection tests against nearby game objects.
6.  **Output:** The methods invoke a consumer callback for each entity or block that successfully passes the collision and line-of-sight tests.

