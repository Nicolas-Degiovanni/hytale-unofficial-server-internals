---
description: Architectural reference for SensorValueProviderWrapper
---

# SensorValueProviderWrapper

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility
**Type:** Decorator Component

## Definition
```java
// Signature
public class SensorValueProviderWrapper extends SensorBase implements IAnnotatedComponentCollection {
```

## Architecture & Concepts
The SensorValueProviderWrapper is a specialized Decorator component within the server-side NPC artificial intelligence framework. Its primary architectural role is to act as a dynamic bridge between an NPC's persistent state, held in a ValueStore component, and the conditional logic of a generic, wrapped Sensor.

This class enables a powerful data-driven design pattern. Instead of creating highly specific sensors with hardcoded values (e.g., a sensor that checks for targets within a 10-meter radius), designers can use a generic sensor (e.g., a sensor that checks for targets within a *configurable* radius) and use this wrapper to feed the radius value from the NPC's own ValueStore at runtime.

This decouples the sensor's logic from its configuration. The wrapped Sensor is entirely unaware that its parameters are being dynamically supplied; it simply receives them as part of its standard evaluation context. The SensorValueProviderWrapper intercepts the evaluation call, injects the necessary data, and then transparently delegates the final logic check to the wrapped component.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the server's NPC asset loading pipeline when processing an NPC's behavior definition (Role). It is instantiated based on a corresponding builder configuration, typically defined in a HOCON or JSON asset file. Direct manual instantiation is an anti-pattern.
- **Scope:** The lifecycle of a SensorValueProviderWrapper is tightly bound to the parent Role object it belongs to. It persists in memory for as long as the NPC's behavior definition is loaded.
- **Destruction:** The object is eligible for garbage collection when its parent Role is unloaded and dereferenced. It does not manage any native resources and has no explicit destruction method. Lifecycle events like *unloaded* are delegated to the wrapped sensor.

## Internal State & Concurrency
- **State:** The component's configuration, including the reference to the wrapped Sensor and the value-to-parameter mappings, is **immutable** after construction. However, the internal ParameterProvider objects it manages contain transient, mutable state. This state is intentionally modified during each call to the *matches* method and should not be considered stable across ticks.

- **Thread Safety:** **This class is not thread-safe.** The NPC behavior system operates on a single-threaded model per world simulation. The internal state of the managed ParameterProviders is mutated during the *matches* call, making concurrent access from multiple threads inherently unsafe and likely to cause race conditions or data corruption. All interactions with this component must be synchronized with the owning world's server tick.

## API Surface
The primary contract is the *matches* method, which evaluates the sensor's condition. Other public methods are primarily for lifecycle delegation or introspection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(N) | Evaluates the sensor. First, it delegates to the wrapped sensor. If successful and *passValues* is true, it reads N values from the entity's ValueStore and injects them into the corresponding parameter providers. Fails if the entity lacks a ValueStore. |
| getSensorInfo() | InfoProvider | O(1) | Returns a wrapped InfoProvider that combines information from the underlying sensor with the dynamic parameter providers managed by this wrapper. |
| loaded(role) | void | O(1) | Lifecycle callback. Delegates the call directly to the wrapped sensor. |
| spawned(role) | void | O(1) | Lifecycle callback. Delegates the call directly to the wrapped sensor. |

## Integration Patterns

### Standard Usage
This component is not intended for direct use in procedural code. It is configured declaratively within NPC asset files. The game engine's behavior tree evaluator invokes the *matches* method during an NPC's update tick.

```java
// CONCEPTUAL: The system invokes this component during an NPC's behavior evaluation.
// A developer does not call this directly.

// Inside the NPC tick loop:
boolean conditionMet = sensorValueProviderWrapper.matches(entityRef, role, dt, store);
if (conditionMet) {
    // The behavior tree proceeds to the next node.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SensorValueProviderWrapper()`. The object's complex internal state and mappings must be configured by the asset loading pipeline from a valid builder definition.
- **State Reliance Across Ticks:** Do not assume the parameter values injected during a *matches* call will persist. The state of the internal ParameterProviders is transient and overwritten on every successful evaluation.
- **Concurrent Access:** Never call methods on this object from an asynchronous task or a separate thread. All interactions must occur on the NPC's primary simulation thread to prevent race conditions.

## Data Pipeline
The component's primary function is to orchestrate a data flow from an NPC's state store to a sensor's input parameters.

> Flow:
> NPC Behavior Tick -> **SensorValueProviderWrapper.matches()** -> Reads from entity's `ValueStore` -> Injects values into internal `ParameterProvider`s -> Delegates `matches()` call to wrapped `Sensor` -> Returns final boolean result

