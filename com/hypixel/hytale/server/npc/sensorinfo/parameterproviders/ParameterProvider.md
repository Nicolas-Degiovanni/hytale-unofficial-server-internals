---
description: Architectural reference for the ParameterProvider interface, a core contract for NPC AI sensor configuration.
---

# ParameterProvider

**Package:** com.hypixel.hytale.server.npc.sensorinfo.parameterproviders
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface ParameterProvider {
```

## Architecture & Concepts
The ParameterProvider interface defines a contract for objects that supply configuration and runtime data to the server-side NPC AI system. It is a fundamental component of the AI Sensor framework, enabling a data-driven approach to NPC behavior.

Architecturally, this interface facilitates a hierarchical or indexed data retrieval pattern. An AI Sensor, such as one for detecting nearby entities or environmental conditions, does not hardcode its operational parameters. Instead, it queries an object implementing ParameterProvider to get the values it needs. The `getParameterProvider(int)` method strongly suggests that providers can be nested, forming a tree-like or composite structure. This allows for complex, reusable AI configurations to be built from smaller, specialized provider components.

This interface is the abstraction layer between the static AI definition (e.g., from a configuration file) and the dynamic runtime state of an NPC's sensory systems.

## Lifecycle & Ownership
As an interface, ParameterProvider does not have a concrete lifecycle. The lifecycle described here applies to the *implementing classes* that fulfill this contract.

- **Creation:** Instances are typically created and assembled by an AI configuration loader or an NPC factory when an NPC's brain is initialized. They are part of the NPC's complete AI component graph.
- **Scope:** The lifetime of a ParameterProvider instance is tightly coupled to the AI Sensor or Behavior that owns it. It persists as long as its parent AI component is active, which is generally the lifetime of the NPC itself.
- **Destruction:** The object is eligible for garbage collection when the parent NPC is despawned and its AI components are de-referenced. The `clear()` method is not for destruction but for state reset during the AI tick.

## Internal State & Concurrency
- **State:** The interface contract implies that implementations will be stateful. The presence of the `clear()` method is a strong indicator that implementing classes manage mutable state that must be reset, likely between AI evaluation ticks or behavior executions.
- **Thread Safety:** **Warning:** Implementations of this interface are not guaranteed to be thread-safe. AI logic for different NPCs may be processed concurrently. Implementors must ensure that any shared or mutable state is handled with appropriate synchronization, or that each instance is confined to a single NPC's update thread. Unsynchronized access can lead to severe AI corruption and server instability.

## API Surface
The public contract is minimal but defines a powerful hierarchical access pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getParameterProvider(int) | ParameterProvider | O(1) to O(log N) | Retrieves a nested provider by its index. Implementations may use an array or map. Callers must handle the possibility of a null return. |
| clear() | void | O(N) | Resets the internal state of the provider and any nested providers. Critical for preventing state leakage between AI ticks. |

## Integration Patterns

### Standard Usage
A component within the AI system, such as a Sensor, will hold a reference to a root ParameterProvider. During its update cycle, it will traverse the provider hierarchy to retrieve the specific configuration it needs for the current tick.

```java
// An AI Sensor retrieves its parameter source from the NPC's AI context
ParameterProvider rootProvider = npc.getAIContext().getParameterProviderFor(this.sensorId);

// Access a specific nested configuration for a sub-task
ParameterProvider visionParams = rootProvider.getParameterProvider(VISION_CONFIG_INDEX);

// At the end of the AI tick, the system may clear the provider's state
rootProvider.clear();
```

### Anti-Patterns (Do NOT do this)
- **Ignoring State Reset:** Failing to call `clear()` on stateful implementations can cause data from a previous AI tick to influence the current one, leading to unpredictable and buggy NPC behavior.
- **Unchecked Nulls:** Assuming `getParameterProvider(int)` will always return a valid object. This method can return null if the index is invalid, and failing to check for this will result in a NullPointerException, potentially crashing the NPC's AI routine.
- **Stateful Singletons:** Implementing this interface as a server-wide singleton is extremely dangerous. The state is specific to a single NPC's AI context and must not be shared globally.

## Data Pipeline
ParameterProvider acts as a source or a node in the AI configuration data pipeline. It does not process a continuous stream of data but rather serves as a structured repository for lookup.

> Flow:
> NPC Configuration Files -> AI Context Initializer -> **ParameterProvider Implementation** -> AI Sensor -> Behavior Tree Node Execution

