---
description: Architectural reference for AmbienceTracker
---

# AmbienceTracker

**Package:** com.hypixel.hytale.builtin.ambience.components
**Type:** Component

## Definition
```java
// Signature
public class AmbienceTracker implements Component<EntityStore> {
```

## Architecture & Concepts
The AmbienceTracker is a data-only component within Hytale's Entity Component System (ECS). Its sole responsibility is to hold state that dictates a specific, overriding musical track for a given region of the world.

Architecturally, this component does not contain any logic. It serves as a state container attached to an `EntityStore`, which typically represents a world or a large-scale region. This design decouples the *state* of ambient music from the *systems* that act upon that state. A separate, higher-level `AmbienceSystem` is responsible for querying `EntityStore` entities for this component, reading the `forcedMusicIndex`, and synchronizing the client's music via network packets.

The component pre-allocates an `UpdateEnvironmentMusic` packet. This is a deliberate performance optimization to prevent repeated object allocation during the game loop, as the packet's state can be updated and reused for transmission to multiple clients.

### Lifecycle & Ownership
- **Creation:** An AmbienceTracker is not instantiated directly. It is added to an `EntityStore` by a controlling system, such as a world generation process, a quest script, or an administrative command, to define a special musical zone. The typical mechanism is `entityStore.addComponent(new AmbienceTracker())`.
- **Scope:** The component's lifetime is strictly bound to the `EntityStore` it is attached to. It persists as long as its parent entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent `EntityStore` is unloaded or destroyed. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The component's state is entirely mutable. Both the `forcedMusicIndex` and the internal `musicPacket` can be modified at runtime. It is designed to be a dynamic flag that game systems can change in response to events.
- **Thread Safety:** This component is **not thread-safe**. As is standard for Hytale's ECS components, it must only be accessed from the main server thread that owns the corresponding `EntityStore`. Unsynchronized access from other threads will lead to race conditions and undefined behavior. All interactions must be scheduled as tasks on the main game loop.

## API Surface
The public API is minimal, focusing exclusively on state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the unique type identifier used to register and query for this component within the ECS. |
| setForcedMusicIndex(int) | void | O(1) | Sets the overriding music track index. This is the primary mutation method. |
| getForcedMusicIndex() | int | O(1) | Retrieves the currently configured music track index. |
| getMusicPacket() | UpdateEnvironmentMusic | O(1) | Returns a reference to the internal, reusable network packet object. |
| clone() | Component | O(1) | Creates a shallow copy of the component, primarily for internal ECS operations. |

## Integration Patterns

### Standard Usage
A controlling system retrieves the component from an `EntityStore` and modifies its state to influence the game world's music.

```java
// In a hypothetical system that manages a boss encounter
EntityStore bossArenaRegion = ...;
AmbienceTracker tracker = bossArenaRegion.getComponent(AmbienceTracker.getComponentType());

if (tracker != null) {
    // When the boss fight begins, force the boss music track
    tracker.setForcedMusicIndex(BOSS_MUSIC_TRACK_ID);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not read `getForcedMusicIndex` once and store the value for later use. Other game systems or scripts may change the component's state at any time. Always query the component for the most current value within the same tick you intend to use it.
- **Packet Misuse:** The object returned by `getMusicPacket` is a shared, mutable instance. Do not hold a reference to it or modify it outside the immediate scope of the responsible `AmbienceSystem`. Modifying its state directly can cause synchronization issues if not handled by the authoritative system.
- **Multi-threaded Access:** It is a critical error to access or modify this component from any thread other than the main server thread. Asynchronous operations that need to change music must schedule a task to run on the main game loop.

## Data Pipeline
The AmbienceTracker is an early step in the data flow for synchronizing world music from the server to the client.

> Flow:
> Game Event (e.g., Script, Player Action) -> Game Logic updates **AmbienceTracker** state -> AmbienceSystem reads state on server tick -> AmbienceSystem updates and dispatches `UpdateEnvironmentMusic` packet -> Client receives packet -> Client Audio Engine changes music track

