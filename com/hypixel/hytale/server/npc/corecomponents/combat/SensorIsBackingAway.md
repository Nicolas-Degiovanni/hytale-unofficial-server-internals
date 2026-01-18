---
description: Architectural reference for SensorIsBackingAway
---

# SensorIsBackingAway

**Package:** com.hypixel.hytale.server.npc.corecomponents.combat
**Type:** Component

## Definition
```java
// Signature
public class SensorIsBackingAway extends SensorBase {
```

## Architecture & Concepts
The SensorIsBackingAway class is a specialized component within the server-side NPC artificial intelligence framework. It functions as a high-level conditional check, or *predicate*, used by more complex AI structures like Behavior Trees or Finite State Machines.

Its primary role is to abstract a specific combat behavior—retreating or "backing away"—into a simple boolean state. Instead of querying low-level physics or position data, the AI system can use this sensor to ask a direct question: "Is the NPC currently in a state of tactical retreat?".

This component is a leaf node in the "Sense" phase of the Sense-Think-Act AI loop. It does not initiate actions but provides critical input to decision-making nodes (e.g., Selectors, Sequences) that determine the NPC's subsequent actions. The actual logic for what constitutes "backing away" is delegated to the NPC's active **Role** component, ensuring that behavior is centralized and context-dependent.

### Lifecycle & Ownership
-   **Creation:** An instance of SensorIsBackingAway is created by its corresponding builder, BuilderSensorIsBackingAway. This process is typically driven by a configuration service that deserializes an NPC's definition from asset files (e.g., JSON). The sensor is instantiated as part of the NPC's complete AI component graph upon the NPC's initialization.
-   **Scope:** The sensor's lifetime is tightly coupled to the parent NPC entity. It persists as long as the NPC is active in the world.
-   **Destruction:** The object is marked for garbage collection when the parent NPC entity is destroyed or unloaded from the world.

## Internal State & Concurrency
-   **State:** This class is stateless. Its evaluation logic is entirely dependent on the arguments passed to the `matches` method, particularly the Role object. It does not cache results or maintain internal state across ticks. Any stateful behavior, such as inversion or cooldowns, is managed by the parent SensorBase class.
-   **Thread Safety:** This component is not thread-safe and must not be accessed concurrently. It is designed to be invoked exclusively from the main server thread during the synchronous entity update tick for the world in which the NPC resides. Accessing it from other threads (e.g., network handlers, asynchronous tasks) will lead to race conditions and unpredictable behavior, as the underlying Role state is not thread-safe.

## API Surface
The public contract is minimal, focusing on the evaluation of the sensor's condition.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Returns true if the NPC's active Role is in a "backing away" state. Also invokes pre-condition checks from SensorBase. |
| getSensorInfo() | InfoProvider | O(1) | Intended for debugging; currently returns null, providing no extended information. |

## Integration Patterns

### Standard Usage
A developer or designer typically does not interact with this class directly in code. Instead, it is declared within an NPC's behavior definition files. The AI system then uses it as a condition to control the flow of a Behavior Tree.

**Conceptual Behavior Tree Definition (YAML/JSON):**
```yaml
- Selector:
    - Sequence:
        - Condition: SensorIsBackingAway
        - Action: StrafeWhileShooting
    - Sequence:
        - Condition: SensorIsTargetInRange
        - Action: AttackMelee
```
In this example, the Behavior Tree will only attempt the StrafeWhileShooting action if the SensorIsBackingAway condition evaluates to true.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new SensorIsBackingAway()`. The component must be constructed via its designated builder, which is typically handled by the engine's configuration loader.
-   **External Invocation:** Do not call the `matches` method from outside the NPC's AI update cycle. The state of the `role` object is only guaranteed to be consistent during that specific phase of the server tick.

## Data Pipeline
This sensor acts as a gate in the flow of AI decision-making. It consumes high-level state and produces a boolean signal that directs subsequent behavior.

> Flow:
> Game State (Target Proximity, NPC Health) -> **Role Component** (State changes to "BackingAway") -> AI Ticker evaluates Behavior Tree -> **SensorIsBackingAway.matches()** -> Returns `true` -> Behavior Tree selects a retreat-related Action (e.g., `ActionStrafe`)

