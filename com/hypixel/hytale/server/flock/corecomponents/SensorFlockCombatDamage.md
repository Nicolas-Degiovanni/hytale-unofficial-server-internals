---
description: Architectural reference for SensorFlockCombatDamage
---

# SensorFlockCombatDamage

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorFlockCombatDamage extends SensorBase {
```

## Architecture & Concepts

SensorFlockCombatDamage is a specialized component within the server-side NPC AI framework, designed to detect and react to combat events affecting a group of entities, known as a Flock. It operates as a conditional node within an AI behavior tree or state machine, answering the question: "Has my flock recently been damaged by an attacker?"

This class acts as a bridge between the low-level combat system, which produces DamageData, and the high-level AI decision-making logic. Its primary architectural function is to abstract the complexity of group damage aggregation into a simple boolean check.

A key feature is the **leaderOnly** flag, configured during construction. This allows AI designers to create sophisticated group behaviors:
*   When **false**, any member of the flock taking damage will trigger the sensor. This is useful for "swarm" or "all for one" behaviors where the entire group retaliates.
*   When **true**, only damage dealt to the designated flock leader will trigger the sensor. This facilitates "protect the leader" or "avenge the alpha" scenarios.

Upon a successful match, the sensor's internal EntityPositionProvider is populated with the location of the most significant attacker. This output is then consumed by other AI components, such as a pathfinding or targeting system, to orchestrate a response.

## Lifecycle & Ownership

-   **Creation:** This object is not meant to be instantiated directly. It is constructed by its corresponding builder, **BuilderSensorFlockCombatDamage**, which is typically invoked by the server's AI configuration loader when parsing NPC behavior assets (e.g., JSON files).
-   **Scope:** The lifetime of a SensorFlockCombatDamage instance is tied to the specific AI **Role** or behavior tree that contains it. It is a short-lived, stateful object that exists only as long as its parent AI state is active for a given NPC.
-   **Destruction:** The object is eligible for garbage collection as soon as the NPC transitions to a different AI state and the owning Role is discarded. It does not manage any native resources and requires no explicit cleanup.

## Internal State & Concurrency

-   **State:** This class is stateful. Its primary mutable state is held within the **positionProvider** field. The result of a call to the matches method directly modifies this internal state, either by populating it with an attacker's location or by clearing it. The **leaderOnly** field is immutable and is fixed at construction time.

-   **Thread Safety:** **WARNING:** This class is not thread-safe and must be considered thread-hostile. It is designed to be exclusively accessed and modified by the main server game loop thread during an NPC's AI tick. Accessing its methods from concurrent threads will lead to severe race conditions, inconsistent state, and potential server instability. All interactions with the underlying FlockPlugin and EntityStore are assumed to be non-atomic and unsynchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | Amortized O(1) | Evaluates if the flock has sustained relevant damage. **Side Effect:** Updates the internal positionProvider on success or clears it on failure. |
| getSensorInfo() | InfoProvider | O(1) | Returns the internal positionProvider, which contains data about the attacker if matches returned true. |

## Integration Patterns

### Standard Usage

This sensor is not intended to be used directly in procedural code. It is designed to be defined declaratively within an NPC's behavior assets and managed by the AI engine. The engine invokes the matches method during the "Sense" phase of the AI update cycle.

```java
// Conceptual usage by the AI engine
// Developer code would not typically call this directly.

// During an NPC's AI tick:
boolean hasFlockBeenDamaged = sensor.matches(npcRef, currentRole, deltaTime, worldStore);

if (hasFlockBeenDamaged) {
    InfoProvider info = sensor.getSensorInfo();
    // Cast to EntityPositionProvider and extract attacker location
    // for use in a "MoveTo" or "Attack" action.
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new SensorFlockCombatDamage()`. This bypasses the mandatory builder pattern, leaving the component in an unconfigured and invalid state. Always define sensors via the appropriate asset format.
-   **State Mismanagement:** Do not call getSensorInfo and assume its data is valid without first checking the boolean result of the matches method. If matches returns false, the internal positionProvider is explicitly cleared and will contain stale or null data.
-   **Asynchronous Access:** Never call matches or getSensorInfo from a separate thread. All interactions must be synchronized with the server's main tick.

## Data Pipeline

The flow of information through this component translates a raw combat event into actionable AI intelligence.

> Flow:
> 1. External entity attacks a Flock member.
> 2. Combat System generates **DamageData**.
> 3. DamageData is associated with the **Flock** object (either for the specific member or the leader).
> 4. AI Engine ticks an NPC in the flock, invoking **SensorFlockCombatDamage.matches()**.
> 5. The sensor queries **FlockPlugin** to retrieve the Flock's shared state.
> 6. It reads the relevant DamageData and identifies the attacker's **Ref<EntityStore>**.
> 7. The attacker's reference is used to populate the internal **EntityPositionProvider**.
> 8. The AI Engine reads the **EntityPositionProvider** via getSensorInfo() to inform subsequent actions (e.g., pathfinding).

