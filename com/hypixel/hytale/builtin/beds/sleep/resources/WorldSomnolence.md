---
description: Architectural reference for WorldSomnolence
---

# WorldSomnolence

**Package:** com.hypixel.hytale.builtin.beds.sleep.resources
**Type:** Resource (World-Scoped)

## Definition
```java
// Signature
public class WorldSomnolence implements Resource<EntityStore> {
```

## Architecture & Concepts
The WorldSomnolence class is a world-scoped **Resource** that functions as a state machine for the global sleep status within a single game world. It does not contain any logic itself; it is a pure data container managed by higher-level sleep systems.

This resource is attached directly to a world's EntityStore, making it a singleton for that specific world instance. It serves as the canonical source of truth for systems that need to know if the world is in a sleeping state, such as the time-of-day system which might accelerate time, or monster spawning systems which might be suppressed. Its state is manipulated exclusively by the server-side logic within the BedsPlugin in response to player actions.

## Lifecycle & Ownership
- **Creation:** WorldSomnolence is not instantiated directly. The engine's resource management system creates an instance and attaches it to an EntityStore when a world is loaded and the BedsPlugin is active for that world.
- **Scope:** The lifecycle of a WorldSomnolence instance is strictly bound to the lifecycle of its parent EntityStore. It persists for the entire duration that a world is loaded in memory.
- **Destruction:** The resource is marked for garbage collection and its state is lost when the corresponding world is unloaded from the server.

## Internal State & Concurrency
- **State:** This object is fundamentally **mutable**. Its primary purpose is to hold and update the single *state* field, which references a WorldSleep object (e.g., Awake, Sleeping). The default state upon creation is WorldSleep.Awake.INSTANCE.

- **Thread Safety:** **WARNING:** This class is **not thread-safe**. It contains no internal synchronization mechanisms such as locks or volatile keywords. All reads via getState and writes via setState must be externally synchronized. It is designed to be accessed exclusively from the main server thread for the world it belongs to. Unsynchronized access from other threads (e.g., networking, asynchronous tasks) will lead to race conditions and undefined behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getResourceType() | static ResourceType | O(1) | Retrieves the globally registered ResourceType for WorldSomnolence. |
| getState() | WorldSleep | O(1) | Returns the current sleep state of the world. |
| setState(state) | void | O(1) | Sets the sleep state of the world. This is the primary mutation point. |
| clone() | Resource | O(1) | Creates a shallow copy of the resource, preserving the current state. |

## Integration Patterns

### Standard Usage
Systems should always retrieve the resource from the current world context for each unit of work (e.g., each server tick). Do not cache references to this resource across ticks, as the world could be unloaded.

```java
// Within a server-side system that has access to the world's EntityStore
EntityStore worldStore = ...;
WorldSomnolence somnolence = worldStore.getResource(WorldSomnolence.getResourceType());

if (somnolence != null) {
    if (shouldWorldSleep()) {
        somnolence.setState(WorldSleep.Sleeping.INSTANCE);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new WorldSomnolence()`. This creates an orphan object that is not registered with any world's EntityStore, rendering it useless to the rest of the engine.
- **Cross-Thread Modification:** Do not call setState from an asynchronous task or network thread without first dispatching the operation to the world's main thread. Doing so is a critical race condition.
- **State Caching:** Do not fetch the resource once and store it in a system field. The resource or the entire world may become invalid. Always re-fetch it from the EntityStore when needed.

## Data Pipeline
WorldSomnolence acts as a state repository in the data flow, not an active processor. It is written to by game logic systems and read by other systems that react to world state changes.

> Flow:
> Player Action (enters bed) -> Network Packet -> Server-side **SleepSystem** -> `setState()` on **WorldSomnolence** -> **TimeOfDaySystem** reads state via `getState()` -> Game time accelerates

