---
description: Architectural reference for PreventItemMerging
---

# PreventItemMerging

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** Singleton / Marker Component

## Definition
```java
// Signature
public class PreventItemMerging implements Component<EntityStore> {
```

## Architecture & Concepts
The PreventItemMerging class is a **Marker Component** within the server-side Entity Component System (ECS). Its sole architectural purpose is to act as a behavioral flag on an item entity. It contains no data and has no internal logic; its mere presence on an entity signals to other game systems that the default item-stacking behavior should be suppressed for that entity.

This component is a classic implementation of the Singleton pattern, ensuring that only one instance exists throughout the server's lifetime. This design is highly efficient, as attaching this component to thousands of entities only involves storing a reference to the same, single object in memory, rather than allocating new memory for each one.

It integrates directly with the server's core entity processing loop, specifically the systems responsible for optimizing the game world by merging nearby item entities into a single stack. These systems will explicitly query for the *absence* of this component before initiating a merge operation.

## Lifecycle & Ownership
-   **Creation:** The single `INSTANCE` is instantiated by the JVM during static class initialization. It is not created by any factory or dependency injection container. For the lifetime of the server, this single instance is the only one that will ever exist.
-   **Scope:** Application-wide. The `INSTANCE` persists for the entire duration of the server process. When an entity is "given" this component, it is merely assigned a reference to this globally-scoped singleton.
-   **Destruction:** The `INSTANCE` is eligible for garbage collection only when its class loader is unloaded, which typically occurs at server shutdown.

## Internal State & Concurrency
-   **State:** This component is **stateless and immutable**. It holds no instance fields and its behavior never changes. The `clone` method reinforces this by returning the static `INSTANCE` rather than creating a new object.
-   **Thread Safety:** This class is inherently thread-safe. As a stateless, immutable singleton, it can be safely accessed and shared across any number of threads without locks or other synchronization primitives. The responsibility for thread safety lies with the systems that read the component's presence from an entity, not the component itself.

## API Surface
The public contract is minimal and primarily consists of static members used for registration and retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | PreventItemMerging | O(1) | The canonical singleton instance. Use this value when adding the component to an entity. |
| CODEC | BuilderCodec | O(1) | The network and persistence codec. Always serializes and deserializes to the singleton INSTANCE. |
| getComponentType() | ComponentType | O(1) | Retrieves the component's globally registered type from the EntityModule. |
| clone() | Component | O(1) | **Warning:** Does not create a new object. Returns the shared singleton INSTANCE. |

## Integration Patterns

### Standard Usage
This component should be attached to an entity to signal that it cannot be merged with other items. Always use the static `INSTANCE` field.

```java
// Assume 'itemEntity' is a valid entity in the world
ComponentType<EntityStore, PreventItemMerging> type = PreventItemMerging.getComponentType();

// Add the component to prevent this entity from merging
itemEntity.addComponent(type, PreventItemMerging.INSTANCE);

// To check if an entity can be merged, query for the component's absence
boolean canMerge = !itemEntity.hasComponent(type);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private and attempting to create an instance via reflection will break the singleton contract and lead to unpredictable behavior in systems that rely on reference equality checks (`== INSTANCE`).
-   **Subclassing:** Do not extend this class. Its design as a stateless marker is fundamental. Adding state in a subclass would violate its architectural purpose.
-   **Stateful Logic:** Do not attempt to add logic to this component. It is a passive data tag. All logic that acts upon this flag must reside in external systems (e.g., an `ItemMergingSystem`).

## Data Pipeline
PreventItemMerging does not process data itself. Instead, it acts as a conditional gate within the entity processing pipeline, altering the flow of logic.

> Flow:
> ItemMergingSystem -> Query for nearby item entities -> For each entity pair, check for component -> **[PreventItemMerging Gate]** -> If component is present on either entity, abort merge -> If component is absent, proceed with merge eligibility calculations.

