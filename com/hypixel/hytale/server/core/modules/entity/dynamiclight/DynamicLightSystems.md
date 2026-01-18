---
description: Architectural reference for DynamicLightSystems
---

# DynamicLightSystems

**Package:** com.hypixel.hytale.server.core.modules.entity.dynamiclight
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class DynamicLightSystems {
    // Contains static nested System classes
}
```

## Architecture & Concepts
The **DynamicLightSystems** class is not an instantiable object but rather a namespace for a collection of related Entity Component Systems (ECS). These systems manage the lifecycle of dynamic lighting attached to entities on the server. They act as the glue between persistent entity data, runtime state, and network replication to clients.

The core concept is the separation of persistent and transient light data:
- **PersistentDynamicLight**: A component that defines a light source as part of an entity's permanent template. This is loaded from storage.
- **DynamicLight**: A runtime component that represents an active light source in the world. This component is what the rendering and networking systems actually use.

The systems within this class ensure that the runtime **DynamicLight** component is correctly created from its persistent counterpart and properly cleaned up and synchronized with clients when it is removed.

### Lifecycle & Ownership
- **Creation:** The nested system classes, **Setup** and **EntityTrackerRemove**, are instantiated by the server's primary ECS scheduler during the server bootstrap or module loading phase. They are not created on-demand during gameplay.
- **Scope:** These systems are singletons within the ECS world, persisting for the entire server session. They are stateless processors that operate on entity data each tick or in response to component change events.
- **Destruction:** The systems are discarded and garbage collected only when the server shuts down and the parent ECS world is destroyed.

## Internal State & Concurrency
- **State:** The systems themselves are stateless. They hold immutable references to **ComponentType** and **Query** objects, which are used to filter entities. All mutable state they operate on is external, contained within the ECS **Store** and **CommandBuffer**.
- **Thread Safety:** These systems are **not thread-safe** and are designed to be executed exclusively on the main server thread by the ECS scheduler. The use of a **CommandBuffer** in **EntityTrackerRemove** is a key pattern to defer state mutations, preventing concurrent modification exceptions and ensuring a deterministic update order. Direct invocation from other threads will lead to world state corruption.

## API Surface
The public API consists of the two nested system classes which conform to the ECS framework's interfaces.

### DynamicLightSystems.Setup
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdd | void | O(1) | Callback executed by the ECS engine when an entity with a **PersistentDynamicLight** is created. It creates and attaches the runtime **DynamicLight** component. |

### DynamicLightSystems.EntityTrackerRemove
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onComponentRemoved | void | O(N) | Callback executed when a **DynamicLight** component is removed from an entity. It iterates through all N viewers of the entity and queues a network packet to remove the light effect on their clients. |

## Integration Patterns

### Standard Usage
A developer does not interact with these systems directly. They are registered with the server's ECS world builder at startup. The engine is then responsible for invoking their logic at the correct time.

```java
// Example of registering these systems during server initialization
WorldBuilder builder = new WorldBuilder();

// The EntityTrackerRemove system requires the Visible component type
ComponentType<EntityStore, EntityTrackerSystems.Visible> visibleType = ...;

builder.addSystem(new DynamicLightSystems.Setup());
builder.addSystem(new DynamicLightSystems.EntityTrackerRemove(visibleType));

// The engine now owns and executes these systems automatically.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation and Invocation:** Never create an instance of these systems and call their methods manually. This bypasses the ECS scheduler, query caching, and thread safety mechanisms, and will corrupt the world state.
- **Stateful Systems:** Do not add mutable fields to these system classes. They are designed to be stateless processors to ensure predictable behavior and prevent side effects between ticks.

## Data Pipeline
These two systems handle different parts of the data flow for dynamic lights.

### Setup System: Hydration Pipeline
The **Setup** system handles the creation of runtime light components from persistent data.

> Flow:
> Entity Loaded from Storage -> Entity has **PersistentDynamicLight** -> ECS Query Match -> **DynamicLightSystems.Setup** -> CommandBuffer adds **DynamicLight** component -> Entity now emits light in the world.

### EntityTrackerRemove System: Network Replication Pipeline
The **EntityTrackerRemove** system handles synchronizing the removal of a light source to all relevant clients.

> Flow:
> Game Logic removes **DynamicLight** component -> ECS Event Triggered -> **DynamicLightSystems.EntityTrackerRemove** -> System gets list of clients viewing the entity -> Queues "Remove Component" network packet for each client -> Client receives packet and destroys the light effect.

