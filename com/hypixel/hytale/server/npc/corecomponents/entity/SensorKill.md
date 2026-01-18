---
description: Architectural reference for SensorKill
---

# SensorKill

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorKill extends SensorBase {
```

## Architecture & Concepts

The SensorKill class is a specialized component within the server-side NPC AI framework. It functions as a conditional node, or a "sensor", in the AI's "Sense-Think-Act" lifecycle. Its sole responsibility is to detect if the NPC it is attached to has successfully killed another entity during the game simulation.

Architecturally, SensorKill acts as a predicate within a larger behavior tree or finite state machine. When the AI controller evaluates its current state, it invokes the SensorKill to check for this specific world event. The sensor can be configured in two modes:

1.  **Specific Target Mode:** It checks for the death of a particular entity that the NPC has "marked" for tracking, referenced via a `targetSlot`. This is used for mission-critical or objective-based AI behaviors.
2.  **General Target Mode:** It checks for the death of *any* entity caused by the NPC.

Upon a successful match, the sensor does more than just return true; it captures the world-space coordinates of where the kill event occurred. This positional data is then exposed through the generic InfoProvider interface, allowing subsequent "Action" nodes in the behavior tree to consume it. For example, an action might use this position to make the NPC walk to the location of its vanquished foe.

## Lifecycle & Ownership

-   **Creation:** A SensorKill instance is never created directly with the `new` keyword. It is instantiated exclusively by the NPC asset loading pipeline via its corresponding builder, BuilderSensorKill. This process occurs when the server parses an NPC's behavior definition from its configuration files at startup or during a hot-reload.
-   **Scope:** The lifetime of a SensorKill object is tightly coupled to the NPC's active Role or behavior configuration. It persists as long as the parent AI behavior is active. If an NPC changes its fundamental behavior (e.g., switching from "Patrol" to "Flee"), the entire behavior tree, including this sensor, is discarded.
-   **Destruction:** The object is managed by the Java garbage collector. It is marked for collection when its parent Role or behavior tree is dereferenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency

-   **State:** This class is highly mutable and stateful, but its state is intentionally volatile. The internal PositionProvider, which stores the location of a detected kill, is cleared at the beginning of every evaluation. It only holds data for the duration of a single AI tick in which its `matches` method returns true. It is not designed for long-term data storage.
-   **Thread Safety:** **This class is not thread-safe.** It is designed with the strict assumption that it will be owned and operated by a single NPC's AI update thread. The `matches` method directly modifies internal state without any locks or synchronization primitives. Concurrent access from multiple threads will result in race conditions and undefined AI behavior. All interactions must be serialized on the owning NPC's AI tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | Amortized O(1) | Evaluates if the NPC has killed an entity matching its criteria. This is the primary entry point, called by the AI engine each tick. It internally modifies state. |
| getSensorInfo() | InfoProvider | O(1) | Returns a provider that exposes the position of the detected kill. This should only be called immediately after `matches` returns true in the same tick. |

## Integration Patterns

### Standard Usage

A developer or designer will not interact with this class directly in Java code. Instead, it is declared declaratively within an NPC's behavior asset files (e.g., JSON or a similar format). The AI engine is responsible for instantiating and invoking it.

The conceptual flow within the engine is as follows:

```java
// PSEUDOCODE: Illustrates how the AI engine uses a sensor.
// This code is conceptual and does not represent a real implementation.

// In the AI behavior tree evaluation...
boolean killConditionMet = sensorKill.matches(npcRef, npcRole, deltaTime, worldStore);

if (killConditionMet) {
    // The condition is true, so execute the associated action.
    // The action can now safely retrieve the data captured by the sensor.
    InfoProvider killInfo = sensorKill.getSensorInfo();
    Vector3d killPosition = ((PositionProvider) killInfo).getTarget();
    
    // Use the position for a subsequent behavior, like moving to the location.
    npc.getNavigationComponent().setTarget(killPosition);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new SensorKill(...)`. The object is fundamentally incomplete without being constructed by its designated builder during the asset loading phase, which correctly resolves the `targetSlot` dependency.
-   **State Caching:** Do not call `getSensorInfo` and store the result for use in a later AI tick. The internal state of the sensor is reset on every call to `matches`. The data is only guaranteed to be valid within the same execution block that `matches` returned true.
-   **Cross-NPC Usage:** Do not share a SensorKill instance between multiple NPCs. Each instance is stateful and tied to the specific `DamageData` component of its owner.

## Data Pipeline

The flow of data concerning a kill event through the AI system is sequential and event-driven.

> Flow:
> Entity Death Event -> Combat System -> **NPCEntity.DamageData** -> AI Tick -> **SensorKill.matches()** -> Behavior Tree Action -> **SensorKill.getSensorInfo()** -> NPC Navigation Component

