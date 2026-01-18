---
description: Architectural reference for Timers
---

# Timers

**Package:** com.hypixel.hytale.server.npc.components
**Type:** Transient Data Component

## Definition
```java
// Signature
public class Timers implements Component<EntityStore> {
```

## Architecture & Concepts
The Timers class is a data component within the server-side Entity-Component-System (ECS) architecture, specifically designed for Non-Player Characters (NPCs). Its sole purpose is to act as a container for an array of Tickable objects.

This component strictly adheres to the ECS principle of separating data from logic. It does not contain any logic for updating or processing the timers it holds. Instead, a dedicated system, likely an NPCTickSystem or a similar scheduler, is responsible for querying entities that possess this component. During the server game loop, this system iterates through the Tickable objects and invokes their tick methods, driving time-based NPC behaviors.

In essence, Timers is a data bag attached to an entity, signaling to the rest of the engine that the entity has time-sensitive behaviors that need to be processed on each tick.

### Lifecycle & Ownership
- **Creation:** An instance of Timers is typically created and attached to an NPC entity when a behavior requiring time-based triggers is added. This is managed by higher-level NPC logic, such as behavior trees or state machines, not by direct instantiation in general game code.
- **Scope:** The lifecycle of a Timers component is strictly bound to the lifecycle of the entity it is attached to. It persists as long as the parent NPC exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when the component is explicitly removed from the entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state consists of a single final field, `timers`, which is an array of Tickable objects. While the array reference itself cannot be changed after construction, the Tickable objects within the array are mutable, as they must track their own internal timing state.

- **Thread Safety:** This component is **not thread-safe** and must not be considered so. It is designed to be accessed and managed exclusively by the main server thread that processes the game tick. Unsynchronized access from other threads will lead to race conditions and unpredictable behavior.

> **Warning: Shallow Copy Implementation**
> The `clone()` method performs a shallow copy. It creates a new Timers instance but reuses the *exact same* array of Tickable objects. Modifying a timer in a cloned component will affect the original, and vice-versa. This can lead to severe and difficult-to-diagnose bugs if two systems believe they have an independent set of timers.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the NPCPlugin registry. |
| getTimers() | Tickable[] | O(1) | Returns the internal array of Tickable objects. |
| clone() | Component | O(1) | Creates a shallow copy of this component. **Warning:** The underlying timer array is shared. |

## Integration Patterns

### Standard Usage
A behavior system retrieves this component from an entity to inspect or manage its timers. The system does not hold a long-term reference to the component.

```java
// A hypothetical NPC behavior system
void processWanderBehavior(Entity entity) {
    // Retrieve the component for the current scope
    Timers timersComponent = entity.getComponent(Timers.getComponentType());

    if (timersComponent != null) {
        Tickable wanderTimer = timersComponent.getTimers()[0];
        // Logic to check if the wanderTimer has expired
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new Timers()`. Components must be added to entities via the world or entity manager APIs to ensure they are correctly registered with the ECS.
- **State Sharing via Clone:** Do not rely on `clone()` to create an independent set of timers. The resulting component will share its internal array with the original, leading to shared state bugs.

## Data Pipeline
The Timers component serves as a data source for the server's main ticking loop. It does not process data itself.

> Flow:
> Server Tick Loop -> NPCTickSystem -> Entity Query for **Timers** -> `getTimers()` -> `Tickable.tick()` -> NPC Behavior State Change or Event Fired

