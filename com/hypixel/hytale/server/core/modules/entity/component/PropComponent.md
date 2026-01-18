---
description: Architectural reference for PropComponent
---

# PropComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class PropComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The PropComponent is a server-side **Tag Component** within Hytale's Entity Component System (ECS). Its sole architectural purpose is to mark an entity as a "prop"—a static, non-interactive, or decorative element within the game world. It contains no data and no logic; its mere presence on an entity is a signal to other engine systems.

Systems such as the AI, physics, or combat modules can query for the absence of this component to filter out entities that do not require complex processing. This makes it a critical component for server performance optimization, allowing the engine to bypass costly updates for thousands of decorative objects in a scene.

It is implemented as a stateless Singleton, as there is no need for unique instances or data for each entity it is attached to. Every entity marked as a prop shares the exact same global PropComponent instance.

### Lifecycle & Ownership
- **Creation:** The single `INSTANCE` is instantiated statically by the Java Virtual Machine during class loading. This occurs once when the server starts, long before any worlds are loaded or entities are created.
- **Scope:** The component exists for the entire lifetime of the server process. It is a global, static object.
- **Destruction:** The instance is destroyed only when the server application is shut down and the JVM garbage collects the class loader.

## Internal State & Concurrency
- **State:** The PropComponent is **stateless and immutable**. It contains no member fields and its behavior never changes.
- **Thread Safety:** This class is inherently thread-safe. As a stateless singleton, it can be safely accessed and referenced by any system on any thread without locks or synchronization primitives. The `clone` method returns the singleton instance itself, preventing unnecessary allocations and reinforcing its shared, immutable nature.

## API Surface
The public API is designed for integration with the ECS framework, not for direct invocation by game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type identifier for this component from the central EntityModule. This is the primary mechanism for adding or querying for this component. |
| get() | static PropComponent | O(1) | Returns the global singleton instance of the component. |
| clone() | PropComponent | O(1) | Returns `this`. Fulfills the Component interface contract while enforcing the singleton pattern. |

## Integration Patterns

### Standard Usage
Developers should never interact with the PropComponent instance directly. Instead, they should use the entity's API to add or check for the component's type. This pattern ensures decoupling and proper integration with the ECS framework.

```java
// CORRECT: Adding the component to an entity to mark it as a prop.
// The entity instance is acquired from the world or entity manager.

Entity myDecorativeStatue = world.createEntity();
myDecorativeStatue.addComponent(PropComponent.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new PropComponent()`. The class is a singleton and is not designed for multiple instances. While not prevented by a private constructor in this case, it violates the core design.
- **State Expectation:** Do not attempt to store state in or extend this component. It is a tag, and its purpose is fulfilled by its existence alone. Adding state would break the performance and design assumptions made by other engine systems.
- **Direct Reference:** Avoid using `PropComponent.get()`. Always prefer using the type system via `PropComponent.getComponentType()` for ECS operations.

## Data Pipeline
The PropComponent does not process data; it directs the flow of data processing for an entity. Its presence acts as a filter condition at the start of various system update loops.

> Flow:
> World Data Deserialization -> Entity is created -> **PropComponent** is attached -> AI Update Loop -> Queries for entities *without* PropComponent -> The entity is skipped -> Physics Update Loop -> Queries for entities *without* PropComponent -> The entity is skipped

