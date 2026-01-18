---
description: Architectural reference for the Interactable component.
---

# Interactable

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Singleton

## Definition
```java
// Signature
public class Interactable implements Component<EntityStore> {
```

## Architecture & Concepts
The Interactable component is a server-side **Tag Component** within the engine's Entity-Component-System (ECS) architecture. Its sole purpose is to mark an entity as being capable of interaction by other game systems, such as player actions or AI behaviors.

As a tag component, it contains no data or state. Its mere presence on an entity is the signal. This is a highly efficient pattern that allows systems to quickly query for all interactable entities without needing to read or process any component data.

This component is managed by the EntityModule, which acts as the central registry for component types on the server. The static method getComponentType is the designated entry point for retrieving the component's unique type identifier, which is essential for ECS operations like adding, removing, or querying for components.

## Lifecycle & Ownership
- **Creation:** A single, static instance named INSTANCE is created by the JVM during class loading. This is a true Singleton pattern. The private constructor prevents any other instantiation.
- **Scope:** The component has a global, application-wide scope. The single INSTANCE object persists for the entire lifetime of the server process. When an entity is made "interactable", it receives a *reference* to this shared instance, not a new object.
- **Destruction:** The INSTANCE is eligible for garbage collection only when its class loader is unloaded, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** This component is **stateless and immutable**. It has no instance fields and its behavior never changes.
- **Thread Safety:** The Interactable component is inherently **thread-safe**. Its immutability guarantees that it can be safely accessed, shared, and referenced by any number of entities and systems across multiple threads without requiring any locks or synchronization.

## API Surface
The public contract is minimal, focusing on identity and registration within the ECS framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | ComponentType | O(1) | **STATIC**. Retrieves the registered type identifier for this component from the central EntityModule. This is the primary method for integration. |
| clone() | Component | O(1) | Returns the shared singleton INSTANCE. This overrides the default cloning behavior to enforce the singleton pattern and prevent object proliferation. |

## Integration Patterns

### Standard Usage
Developers must not create instances of Interactable. Instead, they should use its static type accessor to add the component to an entity, typically through an entity builder or a world management API.

```java
// Correctly adding the Interactable component to an entity
// This enables the InteractionSystem to target this entity.

Entity entity = world.createEntity();
entity.addComponent(Interactable.getComponentType());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private and must not be bypassed with reflection. Doing so would violate the singleton contract and lead to unpredictable behavior in systems that rely on reference equality.
- **Adding State:** Do not modify this class to include fields or state. This would break the tag component pattern, causing all entities with this component to unexpectedly share the new state, which is a critical design flaw.
- **Subclassing:** This class is not designed for extension. Its identity is its function.

## Data Pipeline
The Interactable component does not process data itself. Instead, it acts as a data flag that enables other systems to initiate a data or event pipeline.

> Flow:
> Entity Definition (e.g., JSON) -> CODEC Deserializer -> Entity in EntityStore -> **Query for Interactable Component** -> InteractionSystem -> InteractionEvent on Event Bus

