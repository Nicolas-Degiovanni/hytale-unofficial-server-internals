---
description: Architectural reference for DynamicLight
---

# DynamicLight

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class DynamicLight implements Component<EntityStore> {
```

## Architecture & Concepts
The DynamicLight class is a data component within Hytale's server-side Entity-Component-System (ECS) architecture. It does not contain any logic; its sole responsibility is to represent a source of colored light that is attached to and moves with an entity. Examples include torches held by players, glowing projectiles, or magical auras.

This component is fundamental to the server's lighting and rendering pipeline. It acts as a data vessel that game logic systems can manipulate. The server's core engine systems, particularly the network synchronization and world management layers, read this data to update the game state for all connected clients.

A critical architectural feature is the **isNetworkOutdated** flag. This is a "dirty flag" optimization pattern. Instead of sending lighting updates for every entity on every server tick, the network system only synchronizes entities whose DynamicLight component has been marked as outdated. This dramatically reduces network bandwidth by ensuring that lighting packets are only sent when a change has actually occurred.

## Lifecycle & Ownership
- **Creation:** A DynamicLight component is never instantiated directly by game logic. It is created and attached to a server-side entity via the entity's component management API, such as `entity.addComponent(new DynamicLight(...))`. This is typically done by a system responsible for game events, like a player equipping a torch or casting a spell.

- **Scope:** The lifecycle of a DynamicLight instance is strictly bound to the entity it is attached to. It persists as long as the entity exists in the world and retains the component.

- **Destruction:** The component is marked for garbage collection and destroyed when it is explicitly removed from its parent entity or when the entity itself is despawned from the world. There is no manual memory management required.

## Internal State & Concurrency
- **State:** The component's state is mutable. It primarily consists of a ColorLight object, which encapsulates color and intensity, and the `isNetworkOutdated` boolean flag. Any modification to the light properties will also mutate the flag.

- **Thread Safety:** **This component is not thread-safe.** It is designed to be accessed and modified exclusively by the main server game loop thread. Any concurrent modification from asynchronous tasks or other threads will lead to severe race conditions, especially with the `consumeNetworkOutdated` method, potentially causing network desynchronization or corrupted state. All interactions must be queued and executed on the main server tick.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique, static type identifier for this component from the central EntityModule registry. |
| getColorLight() | ColorLight | O(1) | Returns a reference to the internal ColorLight object. **Warning:** Modifying the returned object directly will not trigger the network dirty flag. |
| setColorLight(ColorLight) | void | O(1) | Replaces the internal ColorLight object and critically marks the component as "dirty," scheduling it for network synchronization. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. Returns true if the component was dirty, then sets the internal flag to false. This is intended for exclusive use by the network synchronization system. |
| clone() | Component | O(1) | Creates a deep copy of the component, including a new instance of the ColorLight object. This is essential for entity duplication and state snapshotting. |

## Integration Patterns

### Standard Usage
Game logic should retrieve the component from an entity and use the `setColorLight` method to ensure the network dirty flag is correctly set.

```java
// A system that updates an entity's light based on game state
// Assume 'entity' is a valid server-side entity instance.

entity.getComponent(DynamicLight.class).ifPresent(lightComponent -> {
    // Create a new light configuration
    ColorLight torchLight = new ColorLight(255, 140, 0, 15); // Bright orange torch

    // This call correctly updates the light and marks it for network sync
    lightComponent.setColorLight(torchLight);
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new DynamicLight()` for general use. Components must be managed through the entity's API, for example `entity.addComponent(...)`, to ensure they are correctly registered with the ECS.

- **Mutating the Getter Result:** Modifying the object returned by `getColorLight` is a critical error. The `isNetworkOutdated` flag will not be set, and the changes will never be synchronized to clients.
    ```java
    // INCORRECT: This change will be invisible to clients
    ColorLight light = lightComponent.getColorLight();
    light.setRed(100); // The network system is unaware of this change
    ```

- **Calling `consumeNetworkOutdated`:** Game logic systems must never call `consumeNetworkOutdated`. This method is part of the contract with the network engine. Calling it prematurely will prevent legitimate changes from being sent to clients.

## Data Pipeline
The flow of data for a lighting change is unidirectional, originating from game logic and terminating at the client's renderer.

> Flow:
> Game Logic System -> `setColorLight()` -> **DynamicLight** (state change, `isNetworkOutdated` = true) -> Server Network System (detects dirty flag via `consumeNetworkOutdated()`) -> Entity Update Packet -> Client Network Handler -> Client-side Entity -> Client Rendering Engine

