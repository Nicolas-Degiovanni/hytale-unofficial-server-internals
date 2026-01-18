---
description: Architectural reference for ModelComponent
---

# ModelComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient

## Definition
```java
// Signature
public class ModelComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ModelComponent is a fundamental data component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to associate a visual representation, defined by a Model asset, with a server-side Entity.

This class acts as a simple data container. It holds a reference to the immutable Model configuration and a mutable dirty flag, isNetworkOutdated, which is critical for the network synchronization subsystem. The component's type and lifecycle are managed centrally by the EntityModule, establishing it as a canonical piece of an entity's state. It does not contain any logic itself; it is purely data to be operated upon by other systems.

### Lifecycle & Ownership
-   **Creation:** A ModelComponent is instantiated and attached to an Entity when that Entity is first created or when its visual appearance is defined by a game system. This is typically handled by entity factories or spawning logic.
-   **Scope:** The lifecycle of a ModelComponent is strictly bound to the Entity it is attached to. It persists as long as the parent Entity exists within the world's EntityStore.
-   **Destruction:** The component is marked for garbage collection and effectively destroyed when its parent Entity is removed from the game world. No manual de-allocation is required.

## Internal State & Concurrency
-   **State:** This component is stateful. It contains an immutable reference to a Model object and a mutable boolean flag, isNetworkOutdated. The flag defaults to true upon creation to ensure the entity's model is synchronized to clients when it first spawns.
-   **Thread Safety:** **Not thread-safe.** The component is designed to be accessed and modified exclusively by the main server game loop thread that owns the parent Entity. The read-then-write operation in consumeNetworkOutdated is not atomic and will cause severe race conditions if accessed from multiple threads. All interactions must be synchronized externally by the owning system.

## API Surface
The public contract is minimal, focusing on data retrieval and state consumption for the networking layer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central EntityModule. |
| getModel() | Model | O(1) | Returns the immutable Model asset associated with the entity. |
| consumeNetworkOutdated() | boolean | O(1) | Returns the current network-outdated status and immediately resets it to false. This is a non-idempotent, one-shot operation. |
| clone() | Component | O(1) | Creates a shallow copy of the component. The internal Model reference is shared, not deep-copied. |

## Integration Patterns

### Standard Usage
The component should always be retrieved from an existing Entity instance. Its state is then typically consumed by a system responsible for network replication.

```java
// In a network synchronization system, iterating over entities:
ModelComponent modelComp = entity.getComponent(ModelComponent.getComponentType());

if (modelComp != null && modelComp.consumeNetworkOutdated()) {
    // This entity's model needs to be sent to clients.
    Model model = modelComp.getModel();
    networkPacket.writeModelData(model.getId());
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ModelComponent()` to modify an entity. To add a component, use the appropriate `entity.addComponent()` method, which correctly registers it with the ECS.
-   **State Polling:** Do not call `consumeNetworkOutdated()` unless your system is responsible for acting on the result. Calling it without processing the data will cause network updates to be missed.
-   **Concurrent Access:** Never read or write to a ModelComponent from a thread other than the main game thread responsible for the parent entity. This will lead to data corruption and desynchronization.

## Data Pipeline
The ModelComponent serves as a data source for the server's network replication pipeline. It does not process data itself but flags its data as "dirty" for other systems to consume.

> Flow:
> Entity Spawner attaches **ModelComponent** -> `isNetworkOutdated` is true -> Network Sync System calls `consumeNetworkOutdated()` -> Packet Builder reads `getModel()` -> Network Packet sent to Client

