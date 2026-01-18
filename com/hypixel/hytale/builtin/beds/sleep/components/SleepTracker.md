---
description: Architectural reference for SleepTracker
---

# SleepTracker

**Package:** com.hypixel.hytale.builtin.beds.sleep.components
**Type:** Component

## Definition
```java
// Signature
public class SleepTracker implements Component<EntityStore> {
```

## Architecture & Concepts
The SleepTracker is a server-side data component responsible for optimizing network traffic related to the world's sleep state. It attaches to an EntityStore, which typically represents a game world, and acts as a stateful cache.

Its core function is to prevent the server from sending redundant network packets. A managing system, such as a hypothetical SleepSystem, calculates the current sleep status (e.g., number of players sleeping, whether time can be skipped) on each tick. It then presents this new state to the SleepTracker. The component compares the new state against the last state it successfully sent to clients. Only if a change is detected will it approve the new state for network transmission.

This component is a key element in the "Beds" feature, ensuring that client UIs and world states are updated efficiently without flooding the network with unchanged data. It embodies a common network optimization pattern: **state caching and delta comparison**.

## Lifecycle & Ownership
- **Creation:** A SleepTracker is not instantiated directly. It is attached to an EntityStore by a higher-level system, likely the BedsPlugin, when a world is initialized or when the sleep feature becomes active for that world. The engine uses the static getComponentType method to manage its identity and attachment.
- **Scope:** The lifecycle of a SleepTracker instance is strictly bound to the EntityStore it is attached to. It persists as long as the world exists on the server.
- **Destruction:** The component is marked for garbage collection and destroyed when its parent EntityStore is unloaded or destroyed. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The component maintains a single piece of mutable state: the `lastSentPacket` field. This field holds a copy of the last UpdateSleepState packet that was generated for network transmission, serving as the cache for comparison. The initial state represents a default "not sleeping" status.
- **Thread Safety:** This component is **not thread-safe**. It is designed to be accessed exclusively from the main server thread that ticks the world (EntityStore) it belongs to. All interactions, particularly calls to generatePacketToSend, must be synchronized with the world's update loop. Unsynchronized access from other threads will lead to race conditions and unpredictable network behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the component's unique type identifier from the BedsPlugin registry. |
| generatePacketToSend(state) | UpdateSleepState | O(1) | Compares the provided state with the last sent state. Returns the state if it has changed, otherwise returns null. |
| clone() | Component | O(1) | Creates a new SleepTracker instance with a default initial state. Used by the engine's component management system. |

## Integration Patterns

### Standard Usage
A server-side system retrieves the SleepTracker from the world's EntityStore, calculates the current sleep state, and uses the component to determine if a network update is necessary.

```java
// Within a server-side system that ticks every world
EntityStore worldEntityStore = ...;
SleepTracker tracker = worldEntityStore.getComponent(SleepTracker.getComponentType());

if (tracker != null) {
    // 1. Calculate the current, real-time sleep state for the world
    UpdateSleepState currentState = calculateWorldSleepState(worldEntityStore);

    // 2. Ask the tracker if this state is new
    UpdateSleepState packetToSend = tracker.generatePacketToSend(currentState);

    // 3. If the packet is not null, a change was detected and it should be sent
    if (packetToSend != null) {
        broadcastPacketToPlayersInWorld(packetToSend);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new SleepTracker()`. Components must be managed by the engine. Always retrieve the component from its parent EntityStore. Failure to do so will result in a disconnected component that does not participate in the world's lifecycle.
- **State Tampering:** Do not access or modify the internal `lastSentPacket` field via reflection or other means. This will corrupt the component's state and break the network optimization logic, potentially causing a flood of packets or a loss of updates.
- **Cross-Thread Access:** Do not read from or write to this component from any thread other than the world's primary update thread. This will cause severe race conditions.

## Data Pipeline
The SleepTracker acts as a gatekeeper in the data flow for sleep status updates.

> Flow:
> World Tick Logic -> Sleep State Calculation -> **SleepTracker.generatePacketToSend** -> (if changed) -> Network Broadcast System -> Client Player

