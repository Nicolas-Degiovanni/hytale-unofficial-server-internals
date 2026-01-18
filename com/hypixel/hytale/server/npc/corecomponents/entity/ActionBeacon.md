---
description: Architectural reference for ActionBeacon
---

# ActionBeacon

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient

## Definition
```java
// Signature
public class ActionBeacon extends ActionBase {
```

## Architecture & Concepts
ActionBeacon is a server-side AI behavior primitive that enables one Non-Player Character (NPC) to broadcast a message to other nearby NPCs. It functions as a localized, spatially-aware communication mechanism within the NPC action system, allowing for emergent group behaviors without direct, hardcoded coupling between AI agents.

This class acts as the *initiator* in a decoupled messaging pattern. The sender (the NPC executing this action) does not know or care how the recipients will react to the message. It is only responsible for identifying valid recipients and dispatching the message payload. Recipients must possess a corresponding **BeaconSupport** component to listen for and process these messages.

The core design principles are:
- **Spatial Awareness:** The action operates within a defined spherical range, leveraging the engine's optimized **PositionCache** for efficient entity lookups.
- **Targeted Filtering:** Broadcasts are not global. They can be filtered by NPC group affiliation, ensuring that, for example, "wolf pack" messages are only received by other wolves.
- **Subject-Oriented Messaging:** A message is not just a string; it is fundamentally linked to a subject entity (the *target*). This allows an NPC to broadcast information *about* another entity, such as "Alert: Hostile player spotted at this location".
- **Delivery Modes:** The system supports both a full broadcast to all valid recipients within range and a stochastic, sampled delivery to a fixed number of random recipients. This is controlled by the `sendCount` parameter.

A notable feature is the extensive in-game debug visualization. When enabled, the system will render arrows from the sender to each recipient, providing a clear, real-time view of the inter-NPC communication flow.

## Lifecycle & Ownership
- **Creation:** ActionBeacon instances are not created directly. They are instantiated by the NPC's configuration loader from a corresponding **BuilderActionBeacon** definition, typically defined in an NPC asset file. An instance is created once per action definition within an NPC's **Role**.
- **Scope:** The object's lifetime is tied to its parent **Role**. It is a persistent, reusable component of the NPC's behavioral repertoire for as long as the NPC entity exists.
- **Destruction:** The instance is marked for garbage collection when the parent NPC entity is unloaded from the world and its **Role** is destroyed.

## Internal State & Concurrency
- **State:** The object's configuration (range, message, target groups) is immutable, established at creation time from its builder. However, it contains one piece of mutable, ephemeral state: the `sendList`. This list is used as a temporary buffer for reservoir sampling when `sendCount` is greater than zero. It is populated and cleared entirely within a single `execute` call.

- **Thread Safety:** **WARNING:** This class is not thread-safe. It is designed to be executed by a single-threaded, per-world AI scheduler. The mutable `sendList` field makes concurrent calls to `execute` on the same instance unsafe, which would result in race conditions and data corruption. All interactions with an ActionBeacon instance must be synchronized by the calling AI system.

## API Surface
The primary contract is defined by its parent, ActionBase. The key public methods are invoked by the NPC's AI scheduler.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| registerWithSupport(Role role) | void | O(1) | A setup hook called during AI initialization. It informs the engine's PositionCache of this action's maximum query range, allowing the cache to be pre-warmed. |
| canExecute(...) | boolean | O(1) | A predicate to determine if the action can run. Checks base conditions (e.g., cooldowns) and verifies that the required target entity exists in the NPC's memory. |
| execute(...) | boolean | O(N) | Triggers the beacon broadcast. Queries for N entities in range, filters them, and dispatches messages. The complexity is proportional to the number of entities within the specified range. |

## Integration Patterns

### Standard Usage
ActionBeacon is not intended for direct developer invocation. It is a component within an NPC's behavior tree or state machine. The engine's AI scheduler is responsible for invoking its methods at the appropriate time during the game tick.

The following conceptual example illustrates how the AI scheduler would drive the action.

```java
// Conceptual code executed by an NPC's AI scheduler
ActionBeacon beaconAction = npc.getRole().findAction(ActionBeacon.class);

// During the AI update tick for a specific NPC (ref)
if (beaconAction.canExecute(ref, role, sensorInfo, dt, store)) {
    // If preconditions are met, execute the broadcast
    beaconAction.execute(ref, role, sensorInfo, dt, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new ActionBeacon()`. Instances must be configured and created via the **BuilderActionBeacon** as part of the asset loading pipeline. Direct creation will result in an unconfigured and non-functional object.
- **Concurrent Execution:** Never call `execute` on the same ActionBeacon instance from multiple threads. The internal `sendList` is not protected and will become corrupted.
- **State Assumption:** Do not read the `sendList` field from outside the class. It is an internal implementation detail, and its contents are only valid during the immediate scope of an `execute` call.

## Data Pipeline
The flow of information for a beacon broadcast is a multi-stage process orchestrated by the `execute` method.

> Flow:
> AI Scheduler Tick -> **ActionBeacon.execute()** -> PositionCache Spatial Query -> Filter by Group & BeaconSupport -> Message Dispatch -> Recipient's BeaconSupport.postMessage() -> Recipient's AI State Update

