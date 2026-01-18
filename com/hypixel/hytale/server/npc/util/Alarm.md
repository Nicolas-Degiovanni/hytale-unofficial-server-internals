---
description: Architectural reference for Alarm
---

# Alarm

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Data Model / Component

## Definition
```java
// Signature
public class Alarm extends PersistentParameter<Instant> {
```

## Architecture & Concepts
The Alarm class is a specialized, persistent data component used within the server-side NPC framework. It is not a standalone system but rather a fundamental building block for creating time-based NPC behaviors. Its primary purpose is to represent a single, future point in time that can be saved and loaded along with an NPC's state.

Architecturally, Alarm is a concrete implementation of the `PersistentParameter` abstraction. This design allows NPC behavior trees, scripts, or AI routines to set a "timer" for a future action and have that timer persist across server restarts, chunk unloading, or other state transitions. The `CODEC` static final field is the key to this persistence, providing the logic for serializing and deserializing the internal `Instant` value to and from the server's storage backend.

An Alarm should be considered a piece of an NPC's memory or internal state, analogous to a variable in a state machine.

### Lifecycle & Ownership
- **Creation:** An Alarm is not instantiated directly. It is declared and managed by the NPC's parameter storage system. Instances are created by the persistence framework during NPC deserialization or initialized as part of an NPC's default state definition.
- **Scope:** The lifecycle of an Alarm instance is strictly tied to its owning NPC entity. It exists as long as the NPC exists in the world.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is destroyed and removed from the game world.

## Internal State & Concurrency
- **State:** The state is mutable and consists of a single nullable field, `alarmInstant`. A null value indicates the alarm is not set. A non-null value represents the specific moment in time the alarm is set for.
- **Thread Safety:** This class is **not thread-safe**. It contains no internal locking mechanisms. All interactions, including setting the time or checking if it has passed, must be performed on the main server thread that is responsible for ticking the owning NPC. Unsynchronized access from other threads will lead to undefined behavior and data corruption.

## API Surface
The primary public contract is defined by its own methods and supplemented by the parent `PersistentParameter` API for setting the value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isSet() | boolean | O(1) | Returns true if the alarm has a configured time. |
| hasPassed(Instant) | boolean | O(1) | Checks if the provided time is after the configured alarm time. This is the primary query method. |

## Integration Patterns

### Standard Usage
The Alarm is typically retrieved from an NPC's state container and checked within the NPC's update or tick logic. The `Instant` passed to `hasPassed` must come from a canonical game clock to ensure consistency.

```java
// Within an NPC's behavior tick method
// Assume 'npc' is the current NPC instance and 'gameClock' is a server-wide time source

// Retrieve the parameter; it's often identified by a string key.
Alarm patrolCooldown = npc.getParameters().get("patrolCooldown", Alarm.class);

if (patrolCooldown.isSet() && patrolCooldown.hasPassed(gameClock.now())) {
    // Cooldown has expired, trigger the next patrol.
    // ... logic to start patrol ...

    // Clear the alarm so this block doesn't run again.
    patrolCooldown.setValue(null);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new Alarm()`. The persistence and state management systems will not be aware of it, and its state will be lost. It must be managed by the NPC's parameter set.
- **Asynchronous Modification:** Do not set or check the alarm from a separate thread (e.g., a network callback or a database worker) without synchronizing back to the main game thread.
- **Manual Time Management:** Avoid using `Instant.now()` directly. Always use a centralized, mockable game clock service provided by the engine. This ensures that time-based logic is consistent and testable, especially if the game simulation can be paused or accelerated.

## Data Pipeline
The Alarm's primary data flow is related to persistence and runtime checks.

**Persistence Flow (Serialization):**
> NPC Entity State -> **Alarm Instance** -> `Alarm.CODEC` -> Serialized `Instant` -> Storage (Disk/Database)

**Runtime Flow (Decision Making):**
> Game Clock Tick -> NPC Behavior Update -> `npc.getParameters().get("key", Alarm.class)` -> **`alarm.hasPassed(gameClock.now())`** -> Boolean Result -> AI State Transition

