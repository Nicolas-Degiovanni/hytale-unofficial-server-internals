---
description: Architectural reference for the Frozen component, a stateless marker in the Entity Component System.
---

# Frozen

**Package:** com.hypixel.hytale.server.core.entity
**Type:** Singleton

## Definition
```java
// Signature
public class Frozen implements Component<EntityStore> {
```

## Architecture & Concepts
The Frozen class is a specialized implementation of the Component interface within Hytale's Entity Component System (ECS). It serves as a **stateless marker component**, also known as a "tag" component. Its sole purpose is to signify that an entity is in a frozen state.

Unlike components that store data (e.g., Health, Position), Frozen contains no fields. Its presence or absence on an entity is the data itself. This is a highly efficient, memory-conscious design pattern. Systems, such as the physics or AI modules, can query for entities that possess this component to alter their behavior accordingly—for example, by halting movement and animations.

It is implemented as a singleton, ensuring that only one instance of Frozen ever exists in the server's memory. All entities marked as frozen share this single, immutable instance, minimizing overhead.

### Lifecycle & Ownership
- **Creation:** The single static instance is created by the Java Virtual Machine during class loading. It is not managed by a dependency injection framework or created on-demand.
- **Scope:** The instance is globally scoped and persists for the entire lifetime of the server application.
- **Destruction:** The instance is destroyed only when the server application is shut down and its class loader is garbage collected.

## Internal State & Concurrency
- **State:** The Frozen component is **immutable and stateless**. It contains no instance fields and its identity is its only relevant attribute.
- **Thread Safety:** This class is inherently thread-safe. As a stateless singleton, it can be safely referenced and used by any system across any thread without locks or synchronization. The core ECS framework is responsible for ensuring thread-safe addition or removal of components from entities.

## API Surface
The public API is minimal, reflecting its role as a static marker. Direct interaction is rare; developers typically interact with it via the entity's component management API.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | Frozen | O(1) | Statically retrieves the global singleton instance. |
| getComponentType() | ComponentType | O(1) | Retrieves the unique type identifier for the Frozen component from the central EntityModule. |
| clone() | Component | O(1) | Returns the singleton instance. This overrides the standard clone behavior to prevent new allocations. |

## Integration Patterns

### Standard Usage
A developer should never instantiate or directly hold a reference to the Frozen object. Instead, they should interact with the ECS to add, remove, or check for the component's presence on an entity.

```java
// Correct: Adding the Frozen state to an entity
Entity targetEntity = ...;
targetEntity.addComponent(Frozen.getComponentType());

// Correct: Checking if an entity is frozen
boolean isFrozen = targetEntity.hasComponent(Frozen.getComponentType());

// Correct: Removing the Frozen state from an entity
targetEntity.removeComponent(Frozen.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to create a new instance via reflection or other means violates the singleton contract and is unsupported.
- **Stateful Subclassing:** Do not extend this class to add state. If you need a "frozen" state with data (e.g., duration), create a new, distinct component class. The purpose of Frozen is to be a lightweight, data-free tag.
- **Instance Comparison:** Do not rely on `frozenComponentA == frozenComponentB` for logic. While it will work due to the singleton pattern, the canonical approach is to check for the component's existence on an entity via its ComponentType.

## Data Pipeline
The Frozen component does not process a data stream. Instead, it acts as a signal within the game loop that alters the data processing pipelines of other systems.

> Flow:
> Game Event (e.g., Ice Spell Hit) -> Gameplay Logic System -> Entity API -> **Frozen component is added to Entity** -> AI & Physics Systems query for entities with Frozen -> Processing is skipped or modified for the tagged entity.

