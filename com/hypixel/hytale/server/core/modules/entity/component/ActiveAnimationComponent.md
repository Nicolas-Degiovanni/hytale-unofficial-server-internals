---
description: Architectural reference for ActiveAnimationComponent
---

# ActiveAnimationComponent

**Package:** com.hypixel.hytale.server.core.modules.entity.component
**Type:** Transient Data Component

## Definition
```java
// Signature
public class ActiveAnimationComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ActiveAnimationComponent is a server-side data component within the core Entity-Component-System (ECS) architecture. Its sole responsibility is to maintain the state of animations currently playing on a single entity. It does not contain any logic for playing, blending, or transitioning animations; it is purely a state container manipulated by other, more complex systems.

State is managed in a fixed-size array, indexed by the AnimationSlot enum. This design provides a predictable structure for an entity's animation channels, such as a main body animation, an upper-body action, or a facial expression.

A critical feature is the **isNetworkOutdated** flag. This is a "dirty flag" that signals to the server's networking layer that the component's state has changed and must be synchronized with clients. This is a fundamental optimization pattern to prevent sending redundant animation data on every network tick, conserving bandwidth. The responsibility for setting this flag lies with the system that modifies the component's state.

## Lifecycle & Ownership
- **Creation:** An ActiveAnimationComponent is never instantiated directly. It is added to an entity by a managing system, typically during entity spawning or when an entity gains the ability to play animations. The clone method facilitates duplication, often used for creating entity archetypes or saving state.
- **Scope:** The lifecycle of this component is strictly bound to the entity it is attached to. It persists as long as the parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is destroyed or when the component is explicitly removed from the entity via the ECS framework.

## Internal State & Concurrency
- **State:** The component's state is entirely mutable. The internal activeAnimations array can be modified at any time through the public API. Each entity possesses a unique instance of this component, ensuring state is not shared.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within the server's main game loop. All reads and writes must be synchronized externally, typically by confining all entity updates to a single thread. The consumeNetworkOutdated method, with its non-atomic read-then-write pattern, is particularly vulnerable to race conditions if accessed concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier for this component from the central EntityModule. |
| getActiveAnimations() | String[] | O(1) | Returns a direct reference to the internal animation state array. **Warning:** Modifying this array directly is an anti-pattern. |
| setPlayingAnimation(slot, animation) | void | O(1) | Sets or clears the animation for a given AnimationSlot. Does not set the network dirty flag. |
| consumeNetworkOutdated() | boolean | O(1) | Atomically reads and resets the network dirty flag. Intended for exclusive use by the network synchronization system. |
| clone() | Component | O(N) | Creates a new component instance with a copy of the current animation state. N is the number of AnimationSlot values. |

## Integration Patterns

### Standard Usage
This component is typically manipulated by a server-side system (like an AnimationSystem or AISystem) and read by the NetworkSystem.

```java
// In an AI or Animation System
// Assume 'entity' is the target entity object
ActiveAnimationComponent anim = entity.getComponent(ActiveAnimationComponent.class);
if (anim != null) {
    anim.setPlayingAnimation(AnimationSlot.MAIN, "walk_cycle");
    // The system MUST then mark the component as dirty for network replication.
    // This is often handled by a separate call, e.g., entity.markComponentDirty(anim);
}

// In the Network System
ActiveAnimationComponent anim = entity.getComponent(ActiveAnimationComponent.class);
if (anim != null && anim.consumeNetworkOutdated()) {
    // Serialize anim.getActiveAnimations() and send to clients.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActiveAnimationComponent()`. Components must be created and managed by the ECS framework, for example, via `entity.addComponent()`.
- **Direct State Mutation:** Do not modify the array returned by getActiveAnimations. This bypasses the component's API and can break assumptions made by other systems.
- **Improper Flag Consumption:** Only the network synchronization layer should ever call consumeNetworkOutdated. Calling it from game logic will cause state changes to be missed by clients, resulting in desynchronization.
- **Concurrent Modification:** Do not access or modify this component from multiple threads without external locking. The internal state is not protected against race conditions.

## Data Pipeline
This component acts as a stateful data source for the networking pipeline.

> Flow:
> Game Logic (e.g., AI System) -> `setPlayingAnimation()` -> **ActiveAnimationComponent** -> Network System reads state via `consumeNetworkOutdated()` -> Network Packet -> Client Rendering Engine

