---
description: Architectural reference for DeployablesSystem
---

# DeployablesSystem

**Package:** com.hypixel.hytale.builtin.deployables.system
**Type:** System Container

## Definition
```java
// Signature
public class DeployablesSystem {
    // Contains nested static System classes
}
```

## Architecture & Concepts

The DeployablesSystem class is not a singular, stateful system but rather a logical container for a suite of related Entity Component Systems (ECS) that collectively manage the lifecycle and behavior of "deployable" entities. These are typically world objects placed by a player or NPC, such as turrets, mines, or other temporary fixtures.

The architecture separates concerns into three distinct, highly-focused systems:

*   **DeployableRegisterer:** A reactive system that hooks into the core entity lifecycle events. Its sole responsibility is to manage the setup and teardown of a deployable entity when it is added to or removed from the world. This includes playing spawn/despawn effects and, most critically, linking the deployable entity to its owner via the DeployableOwnerComponent.

*   **DeployableTicker:** The primary update loop for the deployable entities themselves. On every server tick, this system iterates through all active entities with a DeployableComponent and invokes their per-frame logic. This is where behaviors like target acquisition, firing, or status effect application are executed.

*   **DeployableOwnerTicker:** The primary update loop for the *owners* of deployable entities. This system iterates through entities that have a DeployableOwnerComponent, allowing for logic that manages the collection of deployables, such as enforcing deployment limits or managing shared resources.

This separation ensures that high-frequency ticking logic is decoupled from the more expensive, one-off setup and teardown logic, following standard ECS performance patterns.

## Lifecycle & Ownership

The systems within this container are managed entirely by the Hytale ECS framework. Manual lifecycle management is not supported and will result in engine instability.

*   **Creation:** Instances of DeployableRegisterer, DeployableTicker, and DeployableOwnerTicker are created by the engine's System Scheduler during world initialization. They are discovered and registered automatically based on their class hierarchy.
*   **Scope:** Each system instance has a scope tied to a specific world. They persist for the entire duration that the world is loaded and active.
*   **Destruction:** The systems are destroyed and garbage collected when the world they belong to is unloaded or the server shuts down.

## Internal State & Concurrency

*   **State:** The DeployablesSystem container and its nested systems are fundamentally **stateless**. They do not store any data between ticks. All state is externalized into the Components they operate on, primarily DeployableComponent and DeployableOwnerComponent. This is a core design principle of the ECS architecture, promoting data locality and cache efficiency.

*   **Thread Safety:** These systems are designed to be operated by a single-threaded System Scheduler within a given world's game loop. They are **not thread-safe** and must not be accessed from other threads. All world modifications, such as spawning particles, playing sounds, or modifying components on other entities, are marshaled through the provided CommandBuffer. The CommandBuffer safely queues these operations for execution at a synchronized point in the game tick, preventing race conditions and concurrent modification exceptions.

## API Surface

The public contract of DeployablesSystem is not a traditional API to be called by developers. Instead, its contract is with the ECS engine itself, defined by the methods it overrides from its base classes.

### DeployableRegisterer API

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Defines the component signature this system targets: entities with a DeployableComponent. |
| onEntityAdded(...) | void | O(log N) | Callback executed by the engine when a matching entity is created. Triggers spawn effects and registers the deployable with its owner. |
| onEntityRemove(...) | void | O(log N) | Callback executed by the engine when a matching entity is destroyed. Triggers despawn effects and deregisters the deployable. |

### DeployableTicker & DeployableOwnerTicker API

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Defines the component signature this system targets: entities with a DeployableComponent or DeployableOwnerComponent, respectively. |
| tick(...) | void | O(N) | Executed by the engine every game tick for each matching entity. Delegates the tick logic to the relevant component. |

## Integration Patterns

Interaction with this system is achieved by manipulating components on entities, not by invoking the system classes directly.

### Standard Usage

To create a deployable entity, a developer must add a DeployableComponent to an entity and, on the owner entity, add a DeployableOwnerComponent. The systems will automatically detect and manage these entities.

```java
// Pseudo-code for creating a deployable
// This logic would typically exist within an item's "use" behavior

// Get the player entity who is deploying the object
Ref<EntityStore> ownerRef = ...;

// Ensure the owner has the necessary component to track deployables
if (!ownerRef.getStore().hasComponent(ownerRef, DeployableOwnerComponent.class)) {
    commandBuffer.addComponent(ownerRef, new DeployableOwnerComponent());
}

// Create the new deployable entity
Ref<EntityStore> deployableEntity = EntityFactory.create("my_turret", commandBuffer);

// Add the DeployableComponent, linking it to the owner
DeployableComponent deployableComp = new DeployableComponent(config, ownerRef);
commandBuffer.addComponent(deployableEntity, deployableComp);
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Never create an instance of these systems using `new DeployableTicker()`. The engine's System Scheduler is solely responsible for their creation and lifecycle.
*   **Manual Invocation:** Do not call the `tick` or `onEntityAdded` methods directly. This bypasses the engine's scheduling and concurrency controls, which will corrupt world state and cause unpredictable crashes.
*   **Stateful Systems:** Do not modify the system classes to hold state (e.g., adding member variables to track entities). This violates ECS principles and will not work correctly in a multi-world environment. All state must reside in components.

## Data Pipeline

The systems act as processors in a data flow orchestrated by the ECS engine.

**Entity Creation Pipeline:**
> Entity is created with `DeployableComponent` -> Engine detects component -> **DeployableRegisterer.onEntityAdded** -> CommandBuffer -> SoundUtil & ParticleUtil -> `DeployableOwnerComponent` is updated with new reference

**Entity Update Pipeline (Per Tick):**
> Engine Tick -> System Scheduler -> **DeployableTicker.tick** -> `DeployableComponent.tick` -> Executes game logic (e.g., targeting, firing) -> Changes are written to Components or CommandBuffer

**Entity Destruction Pipeline:**
> Entity is marked for removal -> Engine detects removal -> **DeployableRegisterer.onEntityRemove** -> CommandBuffer -> SoundUtil & ParticleUtil -> `DeployableOwnerComponent` is updated to remove reference

