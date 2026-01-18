---
description: Architectural reference for ISensorEntityCollector
---

# ISensorEntityCollector

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Contract

## Definition
```java
// Signature
public interface ISensorEntityCollector extends RoleStateChange {
```

## Architecture & Concepts
The ISensorEntityCollector is a core contract within the server-side NPC Artificial Intelligence framework. It operates as a specialized data processor, acting as the final stage in an NPC's sensory perception pipeline. Its primary function is to receive, filter, and aggregate entities detected by a parent Sensor component.

This interface embodies the **Strategy Pattern**. A Sensor is responsible for the *what* and *where* of perception (e.g., detecting entities within a 20-block radius), while an implementation of ISensorEntityCollector defines *how* to process and store the results of that detection. This separation of concerns allows for highly reusable sensor logic combined with behavior-specific collection strategies.

By extending RoleStateChange, this component is explicitly tied to the NPC's state machine, known as the Role system. Its lifecycle methods, init and cleanup, are invoked by the Role system as the NPC transitions between behaviors, ensuring that sensory data is always relevant to the current state.

The static DEFAULT field provides a **Null Object Pattern** implementation, which performs no operations. This is a crucial default for sensors that may not require data collection, preventing null pointer exceptions and simplifying sensor configuration.

## Lifecycle & Ownership
-   **Creation:** Implementations are typically instantiated and configured within an NPC's behavior definition files (e.g., JSON or other configuration formats). They are not created directly in code but are part of the NPC's component-based assembly.
-   **Scope:** The lifetime of an ISensorEntityCollector instance is strictly bound to the activation of its owning NPC Role. It is initialized when the Role becomes active and is destroyed when the Role terminates. It does not persist across Role changes.
-   **Destruction:** The cleanup method is the designated destructor hook. It is invoked by the NPC's state machine when its associated Role is deactivated. Implementations **must** use this method to release all held entity references and reset internal state to prevent memory leaks and stale data.

## Internal State & Concurrency
-   **State:** While the interface itself is stateless, all non-trivial implementations are inherently stateful. They are designed to maintain a collection of entity references (via the Ref wrapper) that meet the sensor's criteria. This state is mutable and highly volatile, intended to be rebuilt from scratch on each AI tick.
-   **Thread Safety:** This component is **not thread-safe** and must not be accessed from multiple threads concurrently. It is designed to be operated exclusively by the server's main NPC update thread. All interactions, from init to cleanup, are expected to occur within a single, synchronized AI tick to prevent race conditions with the underlying world state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(ref, role, accessor) | void | O(1) | Prepares the collector for a new session. Called once when the owning Role is activated. |
| collectMatching(ref, target, accessor) | void | O(1) | Processes and stores a reference to an entity that matches the sensor's criteria. |
| collectNonMatching(target, accessor) | void | O(1) | Informs the collector that a previously matched entity is no longer a match. Used for state eviction. |
| terminateOnFirstMatch() | boolean | O(1) | A performance flag. If true, the parent Sensor will stop its search after the first valid entity is found and collected. |
| cleanup() | void | O(N) | Resets all internal state and releases entity references. Called when the owning Role is deactivated. |

## Integration Patterns

### Standard Usage
An ISensorEntityCollector is never used in isolation. It is configured as part of a Sensor component, which is in turn part of an NPC's Role. The system manages its lifecycle automatically. The primary interaction is for other AI components to query the state of the collector implementation to make decisions.

```java
// Hypothetical AI Brain logic querying a collector
// Note: Direct casting is common after retrieving a known component.

Sensor someSensor = npc.getComponent(Sensor.class);
MySpecificCollector collector = (MySpecificCollector) someSensor.getCollector();

// The brain uses the collected data to make a decision
if (collector.hasHostileTarget()) {
    Ref<EntityStore> target = collector.getNearestHostile();
    // ... initiate combat logic
}
```

### Anti-Patterns (Do NOT do this)
-   **State Leakage:** Failing to clear internal collections during the cleanup call is a critical error. This will cause "ghost" entities to persist in the collector's state after a Role change, leading to unpredictable AI behavior and memory leaks.
-   **Ignoring Non-Matching:** Neglecting to implement logic for collectNonMatching will result in the AI continuing to track entities that have left its sensory range, a common source of bugs.
-   **Cross-Tick Caching:** Do not attempt to cache the collector instance or its data outside the scope of the current AI tick. The data it holds is only guaranteed to be valid for the duration of a single update cycle.

## Data Pipeline
The collector is the terminal point of the NPC perception data flow. It transforms a stream of raw entity detections into a structured, queryable state for the AI's decision-making logic.

> Flow:
> World Spatial Query -> Raw Entity List -> Sensor Filtering Logic -> **ISensorEntityCollector** (collectMatching / collectNonMatching) -> AI Brain (Queries Collector State) -> Action Execution

