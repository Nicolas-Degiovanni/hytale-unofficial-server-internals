---
description: Architectural reference for PreventPickup
---

# PreventPickup

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** Singleton

## Definition
```java
// Signature
public class PreventPickup implements Component<EntityStore> {
```

## Architecture & Concepts
PreventPickup is a **marker component** within the server-side Entity-Component-System (ECS) architecture. It contains no data and serves only as a tag. Its presence on an entity signifies that the entity, typically an item, is immune to standard player pickup mechanics.

This component follows the **Flyweight design pattern**. A single, globally shared instance is used for every entity that requires this behavior. This is extremely memory-efficient, as adding this property to thousands of entities consumes no additional memory beyond the single static instance.

Its primary role is to alter the behavior of other systems, such as an `ItemPickupSystem`. Such systems will query for entities and explicitly filter out any that possess the PreventPickup component, effectively changing their processing logic without direct communication.

## Lifecycle & Ownership
- **Creation:** The single `INSTANCE` is instantiated by the Java ClassLoader when the PreventPickup class is first loaded. It is a static final field, guaranteeing only one instance ever exists.
- **Scope:** Application-wide. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The object is garbage collected only when the server's JVM shuts down. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class contains no instance fields. Its purpose is defined entirely by its type.
- **Thread Safety:** **Inherently thread-safe**. As a stateless singleton, it can be safely accessed and attached to entities from any thread without requiring locks or synchronization. The shared `INSTANCE` is effectively a read-only constant.

## API Surface
The public contract is minimal, focusing on identity and registration rather than behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| INSTANCE | PreventPickup | O(1) | The canonical singleton instance of this component. |
| CODEC | BuilderCodec | O(1) | The network codec for serialization. Always resolves to the singleton INSTANCE. |
| getComponentType() | ComponentType | O(1) | Retrieves the unique type identifier for this component from the EntityModule. |
| clone() | Component | O(1) | Returns the singleton INSTANCE. Reinforces that this component cannot be duplicated. |

## Integration Patterns

### Standard Usage
This component should never be instantiated directly. It is attached to an entity using the entity's component management API, typically by retrieving the singleton `INSTANCE`.

```java
// Correct: Attaching the marker component to an entity
// Assume 'world' and 'entityId' are available context
ComponentType<EntityStore, PreventPickup> type = PreventPickup.getComponentType();
world.getEntityManager().addComponent(entityId, type, PreventPickup.INSTANCE);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Attempting to create a new instance via reflection violates the singleton contract and will cause undefined behavior in systems that rely on reference equality (`== INSTANCE`).
- **Stateful Subclassing:** Do not extend this class to add state. Its design is fundamentally stateless. If you need to store data related to pickup prevention (e.g., a timer), create a new, separate component.
- **Instance Checking:** Avoid checking for the component's presence with `instanceof`. Always use the ECS framework's built-in methods like `hasComponent` for performance and correctness.

## Data Pipeline
PreventPickup does not process data. Instead, it acts as a gate in the data processing pipeline of other systems.

> Flow:
> ItemPickupSystem queries for nearby item entities -> System iterates through results -> For each entity, it checks `entity.hasComponent(PreventPickup.getComponentType())` -> If **true**, the entity is skipped and removed from the processing candidate list. -> If **false**, pickup logic proceeds.

