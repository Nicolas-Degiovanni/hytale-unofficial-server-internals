---
description: Architectural reference for AllNPCsLoadedEvent
---

# AllNPCsLoadedEvent

**Package:** com.hypixel.hytale.server.npc
**Type:** Transient Value Object

## Definition
```java
// Signature
public class AllNPCsLoadedEvent implements IEvent<Void> {
```

## Architecture & Concepts
The AllNPCsLoadedEvent is an immutable, point-in-time signal within the server's event-driven architecture. It represents the successful completion of the initial NPC asset loading and definition parsing phase during server startup.

This class serves as a critical synchronization point. It decouples the low-level NPC asset management subsystem from higher-level game logic systems, such as AI managers, quest engines, and world population services. By broadcasting this event, the asset loader informs the rest of the server that it is now safe to query and instantiate NPC entities.

Implementing the IEvent interface marks this class for dispatch via the central server Event Bus. The generic type parameter of Void signifies that listeners are not expected to return any value; this is a one-way notification.

### Lifecycle & Ownership
- **Creation:** This event is instantiated exclusively by the server's core NPC asset loading service at the end of its bootstrap process. It is created precisely once per server session.
- **Scope:** The object's lifetime is ephemeral. It exists only for the duration of its dispatch through the event bus. Once all subscribed listeners have processed it, it becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There are no native resources or explicit cleanup methods associated with this event.

## Internal State & Concurrency
- **State:** The AllNPCsLoadedEvent is **strictly immutable**. The constructor receives maps of NPC data and immediately wraps them in unmodifiable views using Int2ObjectMaps.unmodifiable. The internal fields are final. Once constructed, the state of the event cannot be altered.

- **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that it can be safely read by multiple listeners across different threads without any risk of data corruption or race conditions. No external synchronization is required when accessing its data.

## API Surface
The public API is minimal, providing read-only access to the results of the NPC loading process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAllNPCs() | Int2ObjectMap<BuilderInfo> | O(1) | Returns an unmodifiable map of all NPC definitions discovered, regardless of their load status. |
| getLoadedNPCs() | Int2ObjectMap<BuilderInfo> | O(1) | Returns an unmodifiable map of only the NPC definitions that were successfully loaded and are ready for instantiation. |

## Integration Patterns

### Standard Usage
This event is not intended to be created by client code. Instead, systems subscribe to it to trigger initialization logic that depends on NPC data being available.

```java
// Example: An AI management system listening for the event
import com.hypixel.hytale.event.Subscribe;

public class AIManager {
    private boolean isReady = false;

    @Subscribe
    public void onAllNPCsLoaded(AllNPCsLoadedEvent event) {
        // The server has confirmed NPCs are loaded.
        // It is now safe to begin AI processing and spawning.
        Int2ObjectMap<BuilderInfo> availableNpcs = event.getLoadedNPCs();
        System.out.println("NPCs are ready. " + availableNpcs.size() + " definitions loaded.");
        this.isReady = true;
        // ... proceed with AI system initialization ...
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of AllNPCsLoadedEvent manually. Doing so would inject a false signal into the server, causing dependent systems to initialize prematurely with incomplete or non-existent data, leading to cascading failures.
- **Attempting State Mutation:** The maps returned by getAllNPCs and getLoadedNPCs are unmodifiable. Any attempt to call methods like put or remove on them will result in an UnsupportedOperationException. The design contract of this event guarantees data integrity.

## Data Pipeline
This event is the terminal output of the NPC asset loading pipeline. It does not process data itself but rather signals the completion of that process.

> Flow:
> NPC JSON/Asset Files on Disk -> Server Asset Loader -> NPC Definition Parsing -> **AllNPCsLoadedEvent (Instantiation)** -> Server Event Bus -> Subscribed Systems (AI, Quests, Spawners)

