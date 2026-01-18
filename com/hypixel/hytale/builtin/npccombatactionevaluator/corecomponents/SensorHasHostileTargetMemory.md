---
description: Architectural reference for SensorHasHostileTargetMemory
---

# SensorHasHostileTargetMemory

**Package:** com.hypixel.hytale.builtin.npccombatactionevaluator.corecomponents
**Type:** Transient Component

## Definition
```java
// Signature
public class SensorHasHostileTargetMemory extends SensorBase {
```

## Architecture & Concepts
SensorHasHostileTargetMemory is a fundamental component within the Hytale Non-Player Character (NPC) AI framework. It functions as a predicate, or a conditional check, within the "Sense" phase of the classic "Sense-Think-Act" AI loop.

Its sole responsibility is to determine if an NPC has registered any hostile entities in its short-term memory. It achieves this by querying the TargetMemory component attached to the NPC entity. This sensor acts as a critical trigger for state transitions, enabling an NPC to switch from passive behaviors (like wandering or foraging) to aggressive or defensive ones (like initiating combat or fleeing).

This class is not intended for direct invocation. Instead, it is configured declaratively as part of an NPC's behavior definition, typically within a Role or a more complex behavior tree structure. The server's AI scheduler then evaluates this sensor during each NPC's update cycle to make decisions about which actions or behaviors to execute.

### Lifecycle & Ownership
-   **Creation:** Instantiated via its corresponding builder, BuilderSensorHasHostileTargetMemory. This process occurs when the server loads and parses NPC definition files, creating reusable, stateless sensor objects that are part of an NPC's behavioral template.
-   **Scope:** The object's lifetime is tied to the NPC's Role definition. It is a stateless, shared instance that persists as long as the NPC type is loaded in memory. It is not created on a per-NPC-instance basis.
-   **Destruction:** The object is eligible for garbage collection when its containing Role or behavior definition is unloaded, such as during a server shutdown or a hot-reload of game data.

## Internal State & Concurrency
-   **State:** This class is **immutable and stateless**. It contains no instance fields that store data. All operations are performed on arguments passed into the `matches` method. The static TARGET_MEMORY field is a constant reference to a component type definition.
-   **Thread Safety:** The class is inherently thread-safe. However, its correct operation relies on the thread-safety of the underlying entity-component system (the `Store<EntityStore>` and the `Ref<EntityStore>`). The engine's architecture is expected to guarantee that AI updates for a specific entity are processed in a thread-safe context, preventing race conditions when accessing component data like TargetMemory.

## API Surface
The public contract is minimal, centered on the evaluation logic inherited from SensorBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | Evaluates the condition. Returns true if the entity's TargetMemory component exists and contains at least one known hostile. Returns false otherwise. |
| getSensorInfo() | InfoProvider | O(1) | Provides debugging information. This implementation returns null, indicating no special debug info is available. |

## Integration Patterns

### Standard Usage
This sensor is not called directly. It is configured within an NPC's definition files (likely JSON or a similar format) and instantiated by the server's AI loading systems. The AI runtime engine then invokes the `matches` method.

A conceptual configuration might look like this:

```java
// Conceptual Example: How the AI system would use the builder
// This code would exist within the server's NPC definition loader.

BuilderSensorHasHostileTargetMemory sensorBuilder = new BuilderSensorHasHostileTargetMemory();

// The built sensor is then attached to a behavior rule or state transition.
SensorHasHostileTargetMemory sensor = sensorBuilder.build();

// The engine would later call sensor.matches(...) during an NPC's update tick.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor requires a builder. Attempting to instantiate it directly is not the intended use case and will fail. Always rely on the engine's configuration loaders.
-   **Misconfiguration:** Applying this sensor to an NPC entity that lacks a TargetMemory component will not cause a crash. However, the `matches` method will consistently return false, preventing the NPC from ever entering a combat state based on this trigger. This is a common logical error during AI configuration.

## Data Pipeline
This component acts as a conditional gate in a data flow, not a transformer. It queries state to produce a boolean decision that influences the subsequent AI behavior.

> Flow:
> NPC AI Update Tick -> Behavior Tree Evaluation -> **SensorHasHostileTargetMemory.matches()** -> Read Entity's TargetMemory Component -> Check `knownHostiles` Collection -> Return `true`/`false` -> Behavior Tree Traversal (e.g., Enter Combat Branch)

