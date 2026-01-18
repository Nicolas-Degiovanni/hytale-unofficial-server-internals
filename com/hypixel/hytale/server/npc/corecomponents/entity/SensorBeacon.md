---
description: Architectural reference for SensorBeacon
---

# SensorBeacon

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorBeacon extends SensorBase {
```

## Architecture & Concepts
The SensorBeacon is a fundamental component within the server-side Non-Player Character (NPC) artificial intelligence framework. It functions as a conditional trigger, or "sensor," that allows an NPC's behavior tree or state machine to react to abstract, non-physical signals broadcast by other entities.

Its primary role is to act as the receiver in a simple, range-limited messaging system. One entity can "send a beacon" to another, and this sensor allows the receiving NPC to detect that signal, identify the sender (or a designated target), and make a decision based on it. This facilitates coordinated group behaviors, quest-related triggers, and other complex interactions without relying solely on direct line-of-sight or proximity detection.

Architecturally, SensorBeacon serves as a bridge between two other components:
1.  **BeaconSupport:** The component attached to an NPC that manages its incoming message queue.
2.  **Role:** The central "brain" of the NPC that orchestrates its behaviors and maintains its short-term memory (Marked Entities).

When polled by the Role, the SensorBeacon checks the BeaconSupport queue for a specific message type. If found, it validates the message's associated target against a distance constraint and, upon success, promotes this target into the Role's memory for use by other AI components.

## Lifecycle & Ownership
-   **Creation:** SensorBeacon instances are not created programmatically during gameplay. They are instantiated by the server's asset loading pipeline when an NPC's definition is loaded from configuration files (e.g., JSON or HOCON). The `BuilderSensorBeacon` class is responsible for parsing the configuration and constructing the SensorBeacon with its immutable parameters like range and message index.
-   **Scope:** The lifecycle of a SensorBeacon is strictly tied to the lifecycle of the parent NPC entity to which it belongs. It persists as long as the NPC is active in the world.
-   **Destruction:** The component is marked for garbage collection when the parent NPC entity is despawned or unloaded from the world. There is no manual destruction method.

## Internal State & Concurrency
-   **State:** This component is stateful. Its primary internal state is managed by the `EntityPositionProvider` field, which caches the reference to a potential target entity discovered during a `matches` call. This state is highly volatile and is cleared or updated on every invocation of the `matches` method.
-   **Thread Safety:** **This class is not thread-safe.** It is designed to be exclusively accessed and manipulated by the server's main game loop thread. It reads from and writes to the shared Entity-Component-System (ECS) data store, which is not protected for concurrent access. Any attempt to invoke its methods from an asynchronous task or worker thread will result in data corruption, race conditions, and server instability.

## API Surface
The public contract is minimal, as the component is primarily driven by the internal AI system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | The core evaluation method. Polls for a beacon message, validates the target's range, and updates the NPC's Role. Returns true if a valid beacon is detected. |
| getSensorInfo() | InfoProvider | O(1) | Provides access to the cached target information from the last successful `matches` evaluation. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke SensorBeacon methods directly. Instead, it is configured within an NPC's asset definition file and is automatically polled by the NPC's `Role` component. The primary interaction is to check the result of the sensor's operation by querying the `Role` for a marked entity.

```java
// Conceptual code within an NPC's behavior logic after the Role has ticked.
// The TARGET_SLOT must match the 'targetSlot' configured for the SensorBeacon.
private static final int TARGET_SLOT = 0;

// ... inside a behavior's update method ...

// The SensorBeacon has already run as part of the Role's update cycle.
// We now check its output, which is stored in the Role's memory.
Ref<EntityStore> beaconTarget = role.getMarkedEntitySupport().getMarkedEntity(TARGET_SLOT);

if (beaconTarget != null && beaconTarget.isValid()) {
    // A valid beacon was detected this tick.
    // The NPC can now act upon the target entity.
    initiateFollowBehavior(beaconTarget);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new SensorBeacon()`. The component requires configuration data passed via its builder during the asset loading phase. Direct instantiation will result in a non-functional component with default or null parameters.
-   **External State Polling:** Do not call `getSensorInfo` in a loop to check for a target. The state is only guaranteed to be valid for the duration of the tick in which `matches` returned true. The canonical pattern is to let the `Role` call `matches` and then check the `MarkedEntitySupport` for the result.
-   **Modifying BeaconSupport Directly:** While another system sends a beacon by modifying `BeaconSupport`, do not attempt to manually poll or consume messages from `BeaconSupport` on an NPC that uses a SensorBeacon. This will interfere with the sensor's logic and cause it to miss triggers.

## Data Pipeline
The flow of information for a beacon signal is linear and crosses multiple components, with SensorBeacon acting as the validation and promotion step.

> Flow:
> Signaling Entity -> Invokes `BeaconSupport.queueMessage` on Target NPC -> Message (`Ref<EntityStore>`) is enqueued in Target NPC's `BeaconSupport` component -> Server Tick -> Target NPC's `Role` component invokes `matches` on its **SensorBeacon** -> **SensorBeacon** reads the message from `BeaconSupport` -> **SensorBeacon** fetches `TransformComponent` for itself and the target -> **SensorBeacon** performs distance check -> If valid, **SensorBeacon** writes target `Ref` to `Role.getMarkedEntitySupport()` -> AI Behavior reads from `Role.getMarkedEntitySupport()` to make a decision.

