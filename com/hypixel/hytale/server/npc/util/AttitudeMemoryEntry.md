---
description: Architectural reference for AttitudeMemoryEntry
---

# AttitudeMemoryEntry

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AttitudeMemoryEntry implements Tickable {
```

## Architecture & Concepts
The AttitudeMemoryEntry is a stateful, self-contained data object that represents a temporary, time-limited override of an NPC's default attitude towards other entities. It is a fundamental component of the server-side NPC AI system, enabling dynamic and reactive behaviors.

This class is not a standalone service but rather a building block used by higher-level AI and memory systems. Its implementation of the **Tickable** interface is the central architectural feature. This contract allows an AttitudeMemoryEntry to be registered with the server's main game loop, which drives its internal state decay. Each entry acts as a "behavioral timer," counting down until its influence expires.

For example, if a player attacks a neutral NPC, the AI system might instantiate an AttitudeMemoryEntry with a hostile Attitude and a duration of 60 seconds. This entry is then added to the NPC's memory, causing it to behave aggressively towards that player until the entry expires, at which point its behavior reverts.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively on the server by AI systems in response to game events. For example, a combat behavior system would create an entry when an NPC is damaged, or a faction system might create one when an NPC witnesses an ally being attacked.
- **Scope:** The object's lifetime is explicitly defined by its `initialDuration`. It is owned by a parent component, typically a collection within an NPC's memory or state manager, and persists only as long as its internal timer has not expired.
- **Destruction:** The object is not destroyed directly. The owning system is responsible for polling the `isExpired` method each tick. Once it returns true, the owner must remove its reference to the AttitudeMemoryEntry. The Java Garbage Collector will then reclaim its memory.

## Internal State & Concurrency
- **State:** This object is **mutable**. The `remainingDuration` field is modified on every call to the `tick` method. The `attitudeOverride` and `initialDuration` fields are immutable after construction.
- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. All interactions, especially calls to `tick`, must be synchronized with the server's main game thread. Unsynchronized access from other threads will lead to race conditions on the `remainingDuration` state and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float dt) | void | O(1) | Decrements the remaining duration by the delta time. This is the primary state mutation method and should only be called by the owning ticking system. |
| getRemainingDuration() | double | O(1) | Returns the time in seconds until this entry expires. |
| getInitialDuration() | double | O(1) | Returns the original duration this entry was created with. |
| getAttitudeOverride() | Attitude | O(1) | Returns the specific Attitude this entry represents (e.g., Hostile, Friendly). |
| isExpired() | boolean | O(1) | Returns true if the remaining duration is less than or equal to zero. This is the primary signal for the owning system to discard this object. |

## Integration Patterns

### Standard Usage
An AttitudeMemoryEntry is managed by a parent NPC controller or memory system. The parent is responsible for ticking the entry and removing it upon expiration.

```java
// Within a hypothetical NPCMemory class
List<AttitudeMemoryEntry> attitudeOverrides = new ArrayList<>();

// On a game event, a new entry is created and added
public void onAttackedBy(Entity attacker) {
    Attitude hostile = AttitudeRegistry.get("hostile");
    AttitudeMemoryEntry entry = new AttitudeMemoryEntry(hostile, 30.0); // Hostile for 30s
    this.attitudeOverrides.add(entry);
}

// During the NPC's main update tick
public void tick(float dt) {
    // Tick all entries and remove expired ones
    attitudeOverrides.forEach(entry -> entry.tick(dt));
    attitudeOverrides.removeIf(AttitudeMemoryEntry::isExpired);
}
```

### Anti-Patterns (Do NOT do this)
- **State Tampering:** Do not attempt to modify the remaining duration from outside the object. The `tick` method is the sole authority for state decay.
- **Re-use:** Do not re-use an expired entry by "resetting" its timer. These objects are designed to be single-use and disposable. Create a new instance for each new behavioral override.
- **Manual Ticking:** Do not call `tick` from arbitrary application logic. It must be driven consistently by the main game loop to ensure accurate timekeeping.

## Data Pipeline
This class functions as a stateful node in the NPC behavior pipeline rather than a data processor. Its lifecycle represents the flow of a temporary state.

> Flow:
> Game Event (e.g., Player Action) -> AI Behavior System -> **new AttitudeMemoryEntry(...)** -> Stored in NPC Memory -> Server Game Loop invokes `tick()` -> `isExpired()` becomes true -> Entry is removed by NPC Memory -> NPC behavior reverts to default.

