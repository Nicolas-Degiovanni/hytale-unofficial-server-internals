---
description: Architectural reference for VoidSpawner
---

# VoidSpawner

**Package:** com.hypixel.hytale.builtin.portals.components.voidevent
**Type:** Data Component

## Definition
```java
// Signature
public class VoidSpawner implements Component<EntityStore> {
```

## Architecture & Concepts
The VoidSpawner is a data-only component within Hytale's Entity-Component-System (ECS) architecture. It serves as a data tag, attaching a list of spawn beacon entity UUIDs to a parent entity. This component does not contain any logic itself; its sole purpose is to hold state that other systems can query and act upon.

Architecturally, it acts as a link between a conceptual "spawner" entity (likely related to a void portal or a dynamic void event) and the physical "spawn point" entities (the beacons). Systems responsible for managing void events will query the world's EntityStore for entities possessing a VoidSpawner component. Upon finding one, these systems will use the stored UUIDs to locate the corresponding beacon entities and execute spawning logic at their positions.

This component is specific to the server-side simulation, as indicated by its association with the EntityStore.

## Lifecycle & Ownership
- **Creation:** A VoidSpawner instance is created and attached to an entity by a higher-level system, typically the PortalsPlugin logic that manages the lifecycle of a void event. It is not intended to be managed manually.
- **Scope:** The lifecycle of a VoidSpawner is strictly bound to the entity to which it is attached. It persists as long as the parent entity exists within the EntityStore.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The component's state is mutable. It primarily consists of an ObjectArrayList of UUIDs, which can be modified after creation.

- **Thread Safety:** This component is **not thread-safe**. The internal list, ObjectArrayList, is not a synchronized collection. All reads and writes to this component's state must be performed on the main server thread to prevent concurrency violations. Unmanaged multi-threaded access will lead to unpredictable behavior and state corruption.

    **WARNING:** The clone method performs a *shallow copy* of the internal list. The list reference is copied, but not the list itself. Modifying the list obtained from a cloned VoidSpawner will also modify the list in the original component. This is a critical detail that can lead to subtle and severe bugs if not handled with extreme care.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the registered type definition for this component from the PortalsPlugin. |
| getSpawnBeaconUuids() | List<UUID> | O(1) | Returns a direct reference to the internal list of spawn beacon UUIDs. |
| clone() | Component<EntityStore> | O(1) | Creates a shallow copy of the component. **Warning:** The internal list is not deep-copied. |

## Integration Patterns

### Standard Usage
A server-side system queries for entities with the VoidSpawner component, retrieves the list of beacon UUIDs, and then resolves those UUIDs to actual entities to perform an action.

```java
// Example within a server-side system
Entity spawnerEntity = ... // An entity known to have a VoidSpawner
VoidSpawner spawner = spawnerEntity.getComponent(VoidSpawner.getComponentType());

if (spawner != null) {
    List<UUID> beaconUuids = spawner.getSpawnBeaconUuids();
    for (UUID beaconUuid : beaconUuids) {
        Entity beacon = world.findEntityByUuid(beaconUuid);
        if (beacon != null) {
            // Execute spawning logic at the beacon's location
            spawnCreaturesNear(beacon.getPosition());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Management Outside ECS:** Do not hold a reference to a VoidSpawner instance outside the scope of its parent entity. Its lifecycle is managed by the ECS, and external references can become stale.
- **Cross-Thread Modification:** Never modify the list returned by getSpawnBeaconUuids from any thread other than the main server thread. This will cause a ConcurrentModificationException or silent data corruption.
- **Ignoring Shallow Clone Behavior:** Do not clone a VoidSpawner and modify its list with the expectation that the original is unaffected. The change will propagate to the original due to the shared list reference.

## Data Pipeline
The VoidSpawner acts as a data source within a larger game logic pipeline. It does not process data itself.

> Flow:
> Void Event System -> Creates Entity & attaches **VoidSpawner** -> Spawning System queries for **VoidSpawner** -> Reads list of UUIDs -> World queries for Entities by UUID -> Spawning logic executes at Entity locations

