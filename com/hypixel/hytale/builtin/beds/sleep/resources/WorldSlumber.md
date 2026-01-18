---
description: Architectural reference for WorldSlumber
---

# WorldSlumber

**Package:** com.hypixel.hytale.builtin.beds.sleep.resources
**Type:** Transient

## Definition
```java
// Signature
public final class WorldSlumber implements WorldSleep {
```

## Architecture & Concepts
The WorldSlumber class is a stateful data model that represents a single, active time-acceleration event within the game world, typically initiated by players sleeping. It is not a service or a manager, but rather a short-lived object that encapsulates the entire state of a "slumber" period.

Its core responsibility is to track the progress of transitioning the in-game time from a defined start instant to a target instant over a specific real-world duration. For example, it might manage the transition from 10:00 PM to 6:00 AM over a period of 5 real-world seconds.

This class acts as a bridge between the server's core time-keeping module and the network protocol. It holds the canonical state for the transition and provides a factory method, createSleepClock, to generate network packets. These packets are broadcast to clients to ensure their visual representation of the skybox and time of day remains synchronized with the server's accelerated clock.

## Lifecycle & Ownership
-   **Creation:** An instance of WorldSlumber is created by a higher-level sleep management system when game logic determines that a time acceleration should begin (e.g., a sufficient number of players are in beds). The parameters for its constructor are supplied by this system based on current world state and game rules.
-   **Scope:** The object's lifetime is strictly bound to the duration of the sleep event it represents. It persists only as long as the time transition is in progress.
-   **Destruction:** The instance is dereferenced and becomes eligible for garbage collection as soon as its internal progress reaches the total duration. The owning sleep management system is responsible for discarding the reference upon completion of the slumber.

## Internal State & Concurrency
-   **State:** WorldSlumber is a **mutable** object. While the start time, target time, and total duration are immutable (set once at construction), the internal `progressSeconds` field is updated on each server tick via the `incProgressSeconds` method. It serves as a live counter for the event's progress.
-   **Thread Safety:** This class is **not thread-safe**. The `incProgressSeconds` method performs a non-atomic read-modify-write operation on its internal state. All interactions with a WorldSlumber instance must be externally synchronized. In practice, it is expected to be exclusively owned and manipulated by the main server game loop thread to prevent race conditions.

## API Surface
The public API is minimal, focusing on state mutation and data transformation for network serialization. Getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| incProgressSeconds(float seconds) | void | O(1) | Advances the slumber's progress by the given real-world time delta. Progress is automatically clamped and will not exceed the total duration. |
| createSleepClock() | SleepClock | O(1) | Creates a network-serializable SleepClock packet. This object contains the start and target times, total duration, and the current progress percentage. |

## Integration Patterns

### Standard Usage
A managing system retrieves the active WorldSlumber instance each server tick, updates its progress with the frame's delta time, and then uses it to generate a synchronization packet for all clients.

```java
// Within a server-side system that runs every tick
WorldSlumber activeSlumber = sleepService.getCurrentSlumber();

if (activeSlumber != null) {
    // Update progress based on time elapsed since last tick
    activeSlumber.incProgressSeconds(tickDeltaTime);

    // Create a packet to synchronize clients
    SleepClock syncPacket = activeSlumber.createSleepClock();
    networkManager.broadcast(syncPacket);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not manually construct a WorldSlumber using `new`. The authority for creating and managing these instances belongs to the server's central sleep or time management system, which ensures the event is triggered by valid game logic.
-   **Concurrent Modification:** Do not access or modify a WorldSlumber instance from any thread other than the main server thread. Doing so will lead to data corruption and desynchronization between the server and clients.
-   **Instance Re-use:** A WorldSlumber object represents a single, continuous time transition. Do not attempt to reset its progress or re-purpose it for a new sleep event. It is designed to be a one-shot object that is discarded after use.

## Data Pipeline
WorldSlumber does not process a stream of data; rather, it generates data based on server ticks. Its primary output is the SleepClock packet, which is fed into the network layer.

> Flow:
> Server Game Tick -> Sleep Management System -> **WorldSlumber.incProgressSeconds()** -> **WorldSlumber.createSleepClock()** -> SleepClock Packet -> Network Subsystem -> Connected Clients

