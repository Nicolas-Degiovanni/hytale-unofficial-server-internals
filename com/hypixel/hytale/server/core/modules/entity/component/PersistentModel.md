---
description: Architectural reference for PersistentModel
---

# PersistentModel

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class PersistentModel implements Component<EntityStore> {
```

## Architecture & Concepts
The **PersistentModel** is a data-only component within the server's Entity-Component-System (ECS) architecture. Its sole responsibility is to associate an entity with a specific visual model definition, represented by a **Model.ModelReference**.

The term *Persistent* in its name is critical; this component is designed for serialization and persistence. It defines the model an entity should have when it is saved to an **EntityStore** (e.g., world data on disk) and subsequently loaded back into the game. It acts as the source of truth for an entity's appearance across game sessions.

Its behavior is governed by the static **CODEC** field, which uses the engine's serialization framework to read and write the model reference to and from a persistent format. This component is a passive data container, manipulated by higher-level systems like the **EntityModule** or world loading services.

## Lifecycle & Ownership
- **Creation:** An instance of **PersistentModel** is created under two primary circumstances:
    1.  **Programmatically:** When a new entity is spawned in the world, game logic instantiates this component with a specific model reference and attaches it to the entity.
    2.  **Deserialization:** When an entity is loaded from an **EntityStore**, the **CODEC** instantiates this component and populates it with the data read from storage.

- **Scope:** The lifecycle of a **PersistentModel** instance is strictly bound to the entity to which it is attached. It exists only as long as its parent entity exists.

- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when the component is explicitly removed from the entity. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of a single field: **modelReference**. This reference can be changed during runtime via the **setModelReference** method to alter an entity's persistent model.

- **Thread Safety:** This class is **not thread-safe**. As a simple data component, it contains no internal synchronization mechanisms.

    **WARNING:** All read and write operations on a **PersistentModel** instance must be performed on the main server thread or be externally synchronized by the system managing the entity. Unsynchronized access from multiple threads will lead to race conditions and unpredictable entity state.

## API Surface
The public API is minimal, focusing on data access and adherence to the **Component** contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the **EntityModule**. |
| getModelReference() | Model.ModelReference | O(1) | Returns the currently assigned model reference. |
| setModelReference(ref) | void | O(1) | Updates the entity's persistent model reference. |
| clone() | Component | O(1) | Creates a shallow copy of the component, duplicating the model reference. |

## Integration Patterns

### Standard Usage
The component is typically instantiated and attached to an entity upon its creation. Game systems then query for this component to perform logic related to persistence or model management.

```java
// Conceptual example of assigning a model to a new entity
Model.ModelReference skeletonModelRef = assetManager.getModel("mob:skeleton");
Entity newSkeleton = world.createEntity();

PersistentModel modelComponent = new PersistentModel(skeletonModelRef);
newSkeleton.addComponent(PersistentModel.getComponentType(), modelComponent);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Instances:** Do not create instances of **PersistentModel** that are not attached to an entity. They serve no purpose and will be garbage collected without effect.
- **Unsynchronized Modification:** Never call **setModelReference** from an asynchronous task or a different thread without acquiring a lock on the parent entity or using a thread-safe command queue. This can corrupt save data.

## Data Pipeline
The primary data flow for this component is during the entity serialization and deserialization process.

> Flow:
> **EntityStore** (Disk Storage) -> Server **CODEC** Framework -> **PersistentModel** (Instantiation & Population) -> Attached to live **Entity** -> Queried by **Game Systems**

