---
description: Architectural reference for the Intangible component.
---

# Intangible

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class Intangible implements Component<EntityStore> {
```

## Architecture & Concepts
The Intangible class is a **Tag Component** within the server-side Entity Component System (ECS). Unlike data components that store state (e.g., Health, Position), a tag component's sole purpose is to mark or "tag" an entity with a specific, binary attribute. Its presence on an entity signifies a state, while its absence signifies the opposite.

In this case, an entity possessing the Intangible component is flagged as non-physical within the game world. This has significant implications for several core systems:
- **Physics & Collision:** The physics engine will ignore entities with this component, allowing them to pass through other entities and solid geometry.
- **Targeting & Combat:** AI and player targeting systems may be configured to disregard intangible entities, rendering them immune to most forms of interaction.
- **Triggers & Volumes:** Intangible entities may not activate trigger volumes.

Architecturally, this class is implemented as a stateless singleton. This is a critical optimization: instead of allocating a new Intangible object for every entity that becomes intangible, the entire server shares a single, static instance. This dramatically reduces memory overhead, especially in scenarios with many intangible entities (e.g., players in a spectator mode).

## Lifecycle & Ownership
- **Creation:** The singleton `INSTANCE` is instantiated once by the JVM during class loading. It is not created on a per-entity basis.
- **Scope:** The singleton object persists for the entire lifetime of the server process. The *association* of this component with a specific entity, however, is transient and managed by the ECS. An entity is only considered intangible for the duration that the component is attached to it.
- **Destruction:** The singleton `INSTANCE` is garbage collected only when the server shuts down. When an entity ceases to be intangible, the ECS simply removes its reference to the shared `INSTANCE`; no object is actually destroyed.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It contains no fields and its identity is constant. Its value is derived purely from its presence or absence on an entity.
- **Thread Safety:** As a stateless singleton, this class is inherently thread-safe. It can be safely referenced and checked from any thread without synchronization. Core ECS operations that add or remove this component from an entity must be synchronized by the managing system, typically the `EntityModule`.

## API Surface
The primary interaction with this component is through the ECS framework, not direct method calls.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central `EntityModule` registry. |
| clone() | Component | O(1) | Returns the singleton `INSTANCE`. This enforces the singleton pattern during entity cloning operations. |

## Integration Patterns

### Standard Usage
Developers should never interact with the Intangible instance directly. Instead, they should use the entity's API to add or remove the component type to modify the entity's state.

```java
// To make an entity intangible
Entity entity = ...;
entity.addComponent(Intangible.getComponentType());

// To check if an entity is intangible
boolean isIntangible = entity.hasComponent(Intangible.getComponentType());

// To make an entity tangible again
entity.removeComponent(Intangible.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to create an instance via reflection would violate the singleton pattern and lead to unpredictable behavior in systems that rely on reference equality (`== INSTANCE`).
- **Stateful Forks:** Do not extend this class to add state. The design is explicitly stateless. If you need to store data related to intangibility (e.g., a duration), create a separate data component.
- **Null Checks:** Do not fetch the component and check for null. The correct, and more performant, pattern is to use `entity.hasComponent()`.

## Data Pipeline
The Intangible component does not process a flow of data. Instead, it acts as a control flag within a larger data processing pipeline, such as the physics simulation.

> Flow:
> Physics Engine Tick -> Iterate Collidable Entities -> **Filter out entities with Intangible component** -> Perform Collision Detection -> Resolve Collisions

