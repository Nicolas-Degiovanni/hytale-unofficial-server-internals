---
description: Architectural reference for PersistentDynamicLight
---

# PersistentDynamicLight

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Data Component

## Definition
```java
// Signature
public class PersistentDynamicLight implements Component<EntityStore> {
```

## Architecture & Concepts
PersistentDynamicLight is a data component within the server-side Entity-Component-System (ECS) architecture. Its sole responsibility is to attach a dynamic, colored light source to an entity. This component directly influences the world's lighting calculations, which are processed on the server and propagated to clients for rendering.

The term *Persistent* is critical; it signifies that the component's state, specifically its ColorLight value, is designed to be serialized and saved to the EntityStore. This ensures that entity-based light sources are restored when a world chunk is loaded from disk. This contrasts with transient, effect-based lights which are not saved.

The static CODEC field defines the binary serialization contract for this component. This codec is used by the engine for two primary purposes:
1.  **Persistence:** Writing to and reading from the world storage (EntityStore).
2.  **Networking:** Synchronizing the light's state with connected clients.

## Lifecycle & Ownership
- **Creation:** An instance is created under two conditions:
    1.  **Programmatic:** A game system attaches a new PersistentDynamicLight to an entity, for example, when a player places a torch item which is then converted into a light-emitting entity.
    2.  **Deserialization:** The EntityStore instantiates the component via its CODEC when loading an entity from the world database.

- **Scope:** The lifecycle of a PersistentDynamicLight instance is strictly bound to its parent entity. It exists only as long as the entity is active in the world.

- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world or when the component is explicitly detached by a game system.

## Internal State & Concurrency
- **State:** The component's state is mutable. It holds a single reference to a ColorLight object, which can be modified at runtime via the setColorLight method. This allows systems to create dynamic lighting effects such as flickering, pulsing, or color changes.

- **Thread Safety:** This component is **not thread-safe**. As with most ECS components, it must only be accessed and mutated from the server's main game loop thread. Unsynchronized, concurrent modification from other threads will lead to data corruption, lighting artifacts, and server instability. All interactions should be scheduled as tasks to be executed on the main thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally unique, registered type identifier for this component class. |
| getColorLight() | ColorLight | O(1) | Returns the current ColorLight state object. |
| setColorLight(ColorLight) | void | O(1) | Updates the component's light state. This will mark the parent entity as dirty for network and storage synchronization. |
| clone() | Component | O(1) | Creates a deep copy of the component, including a new ColorLight instance. Used for entity duplication. |

## Integration Patterns

### Standard Usage
Components should be retrieved from an entity, modified, and left for the engine to synchronize. Never hold a reference to a component outside the scope of a single system update.

```java
// Within a system that updates entities
void updateLight(Entity entity) {
    // Retrieve the component from the entity if it exists
    entity.getComponent(PersistentDynamicLight.class).ifPresent(lightComponent -> {
        // Get the current light state
        ColorLight currentLight = lightComponent.getColorLight();

        // Modify the state based on game logic (e.g., flickering)
        float newIntensity = calculateFlicker(currentLight.getIntensity());
        currentLight.setIntensity(newIntensity);

        // The change is now staged. The engine will handle serialization.
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation for Attachment:** Do not use `new PersistentDynamicLight()` to add a component to an entity. Always use the entity's dedicated API, such as `entity.addComponent()`, which correctly registers the component with the world.

- **External State Caching:** Do not retrieve the ColorLight object and store it in another system for later use. The component's state can be changed by other systems or removed entirely. Always query the component from the entity each time you need to access its state.

- **Cross-Thread Modification:** Never call `setColorLight` from an asynchronous task or a different thread. This will bypass engine state tracking and cause severe concurrency issues.

## Data Pipeline
The data within this component flows from game logic through the engine to storage and connected clients.

> Flow:
> Game Logic (e.g., FlickerSystem) -> **PersistentDynamicLight.setColorLight()** -> Entity marked as dirty -> Server Tick Synchronization -> Component serialized via CODEC -> Data sent to EntityStore (Disk) AND Network Layer (Clients)

