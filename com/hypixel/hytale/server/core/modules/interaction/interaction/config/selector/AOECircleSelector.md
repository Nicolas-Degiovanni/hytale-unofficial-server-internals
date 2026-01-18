---
description: Architectural reference for AOECircleSelector
---

# AOECircleSelector

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.selector
**Type:** Configuration & Factory

## Definition
```java
// Signature
public class AOECircleSelector extends SelectorType {
```

## Architecture & Concepts
The **AOECircleSelector** is a concrete implementation of the *Selector Strategy Pattern*, designed to identify game entities and blocks within a spherical volume. It serves as a data-driven configuration object that defines the properties of an Area-of-Effect (AOE) selection, rather than an active gameplay component.

Its primary architectural role is to bridge the gap between static game data (defined by designers in configuration files) and the dynamic server-side interaction system. The static **CODEC** field is the entry point for this data binding, allowing the engine to deserialize a definition of an AOE circle (e.g., for a spell or weapon) into a usable Java object.

A critical design choice is the separation of configuration from execution. The **AOECircleSelector** class itself holds the immutable parameters of the selection—**range** and **offset**. When an interaction needs to be performed, it calls **newSelector** to acquire a **RuntimeSelector** instance. This inner class contains the actual logic for querying the world via the **CommandBuffer** and is designed to be a lightweight, short-lived object used for the duration of a single selection operation.

This pattern ensures that the core configuration is reusable and thread-safe, while the execution logic is cleanly encapsulated and operates within the context of the server's Entity Component System (ECS).

## Lifecycle & Ownership
-   **Creation:** **AOECircleSelector** instances are not intended for manual instantiation. They are created exclusively by the engine's **Codec** system during the loading and deserialization of game configuration assets, such as item definitions or NPC abilities.
-   **Scope:** An instance persists as long as its parent configuration is loaded in memory. It is effectively a shared, immutable data template. The **RuntimeSelector** instance it vends is transient and should be considered scoped to the single game tick or operation in which it is used.
-   **Destruction:** The object is managed by the Java garbage collector. It is eligible for collection once the asset registry unloads the configuration that contains it.

## Internal State & Concurrency
-   **State:** The state of an **AOECircleSelector** (its **range** and **offset**) is considered immutable after deserialization. Although the fields are not declared final, modifying them at runtime is an unsupported operation that will lead to unpredictable behavior. The vended **RuntimeSelector** is stateless; it processes data passed to it as arguments and does not retain information between method calls.
-   **Thread Safety:** The **AOECircleSelector** is inherently thread-safe for read operations due to its immutable nature. The execution methods within **RuntimeSelector** are **not** thread-safe. They are designed to be called from the main server thread as part of the game tick, operating on a **CommandBuffer**. Accessing these methods from other threads will bypass the server's state management and cause severe concurrency issues, including data corruption and crashes.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| newSelector() | Selector | O(1) | Returns a cached, reusable instance of the runtime selector logic. |
| toPacket() | com.hypixel.hytale.protocol.Selector | O(1) | Serializes the selector's configuration into a network packet for the client. |
| selectTargetPosition(commandBuffer, attackerRef) | Vector3d | O(1) | Calculates the world-space center of the AOE, factoring in the attacker's position, head rotation, and the configured offset. |

## Integration Patterns

### Standard Usage
This class is not used directly by most systems. Instead, an interaction system retrieves the configured selector from a component and uses it to find targets.

```java
// Hypothetical system processing an interaction
void processInteraction(CommandBuffer<EntityStore> commands, Ref<EntityStore> attackerRef) {
    // 1. Retrieve the interaction component from the entity
    InteractionComponent interaction = commands.getComponent(attackerRef, InteractionComponent.class);
    SelectorType selectorConfig = interaction.getSelector(); // This would be an AOECircleSelector instance

    // 2. Obtain the runtime selector logic
    Selector runtimeSelector = selectorConfig.newSelector();

    // 3. Execute the selection logic, providing a consumer for the results
    runtimeSelector.selectTargetEntities(
        commands,
        attackerRef,
        (targetRef, hitLocation) -> {
            // Apply damage or effects to the targetRef
        },
        (targetRef) -> {
            // Predicate to filter targets, e.g., not friendly
            return isHostile(attackerRef, targetRef);
        }
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use **new AOECircleSelector()**. The object will be unconfigured and will cause NullPointerExceptions or incorrect behavior. These objects must only be created via the **CODEC** system from a data source.
-   **State Mutation:** Do not modify the **range** or **offset** fields after the object has been created. This can lead to inconsistent behavior across the server, as the same instance may be shared by multiple systems.
-   **Cross-Thread Execution:** Never call methods on the returned **Selector** instance from an asynchronous task or a different thread. All interaction logic must be executed on the main world thread via a **CommandBuffer**.

## Data Pipeline
The flow of data begins with a designer's configuration file and ends with entities being selected within a game tick.

> Flow:
> Game Asset (JSON/HOCON file) -> Engine **Codec** Deserializer -> **AOECircleSelector** Instance -> **newSelector()** call -> **RuntimeSelector** -> **selectTargetEntities()** -> World query via **EntityStore** -> Target Refs passed to Consumer Lambda

