---
description: Architectural reference for NullSensor
---

# NullSensor

**Package:** com.hypixel.hytale.server.npc.instructions
**Type:** Utility

## Definition
```java
// Signature
public class NullSensor implements Sensor {
```

## Architecture & Concepts
The NullSensor is a server-side component that implements the **Null Object Pattern** for the NPC instruction system. Its primary architectural function is to provide a non-null, default implementation of the Sensor interface that represents an always-satisfied condition.

Within the NPC behavior processing pipeline, Sensors act as conditional gates that determine if an associated instruction should be executed. The NullSensor is a special-case implementation where this gate is permanently open. Its `matches` method unconditionally returns true, ensuring that any behavior or instruction guarded by it is always eligible for execution.

This design simplifies the logic for handling optional or undefined sensor configurations in NPC definitions. Instead of requiring explicit null checks (`if (sensor != null && sensor.matches(...))`), the system can polymorphically call `sensor.matches(...)` on any Sensor instance, including a NullSensor, eliminating the risk of NullPointerExceptions and reducing conditional complexity.

## Lifecycle & Ownership
- **Creation:** The NullSensor is stateless and its behavior is constant. It is intended to be used as a shared, singleton-like instance. It is likely instantiated by a factory or deserializer when an NPC behavior definition omits a specific sensor configuration, or it may be exposed as a static constant (e.g., `Sensor.NONE`).
- **Scope:** A single instance can be shared across the entire server process for its full duration. Its lifetime is not tied to any specific NPC, world, or game session.
- **Destruction:** As a stateless object, it holds no resources. It is eligible for garbage collection only when the server is shutting down and its class loader is unloaded. There is no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** The NullSensor is **stateless and immutable**. It contains no member fields and its methods always return constant values.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable and stateless nature, a single instance can be safely shared and accessed by multiple threads concurrently without any need for locks or synchronization primitives.

## API Surface
The public contract of NullSensor is designed to fulfill the Sensor interface with no-op or default "success" behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(ref, role, dt, store) | boolean | O(1) | **Core Method.** Always returns true, unconditionally satisfying the sensor condition for an instruction. |
| processDelay(dt) | boolean | O(1) | Always returns true, immediately indicating that any time-based delay is complete. |
| isTriggered() | boolean | O(1) | Always returns false. A NullSensor can never be in a "triggered" state. |
| getSensorInfo() | InfoProvider | O(1) | Always returns null. This sensor provides no debug or sensory information. |
| clearOnce() | void | O(1) | No-op. Performs no action. |
| setOnce() | void | O(1) | No-op. Performs no action. |

## Integration Patterns

### Standard Usage
The NullSensor is used in NPC behavior definitions to create instructions that are not gated by environmental or state-based conditions. It effectively makes an action unconditional.

```java
// Conceptual example of an NPC instruction that should always be available
// The system would deserialize a config and substitute NullSensor if none is defined.

Instruction unconditionalInstruction = new Instruction(
    new NullSensor(), // This sensor always returns true
    new FollowPlayerAction()
);

// The behavior processor can now run this without a complex check
if (unconditionalInstruction.getSensor().matches(...)) {
    // This block will always execute
    unconditionalInstruction.getAction().perform();
}
```

### Anti-Patterns (Do NOT do this)
- **Conditional Checks:** Checking if a sensor is an instance of NullSensor completely defeats the purpose of the pattern. Code should never branch based on this type.
    ```java
    // ANTI-PATTERN: This check is redundant and brittle.
    if (sensor instanceof NullSensor || sensor.matches(...)) {
        // ...
    }

    // CORRECT: Rely on polymorphism.
    if (sensor.matches(...)) {
        // ...
    }
    ```
- **Unnecessary Instantiation:** While safe, repeatedly creating new instances (`new NullSensor()`) is inefficient. The system should use a shared static instance to avoid unnecessary object allocation.

## Data Pipeline
The NullSensor does not process data; it is a control-flow component. It acts as a fixed entry point in the decision-making pipeline for an NPC.

> Flow:
> NPC Behavior Processor -> **NullSensor.matches()** -> Returns `true` -> Associated Instruction is executed

