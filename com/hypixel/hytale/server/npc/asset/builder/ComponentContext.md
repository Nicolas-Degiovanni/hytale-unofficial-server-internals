---
description: Architectural reference for ComponentContext
---

# ComponentContext

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Utility / Type-Safe Enum

## Definition
```java
// Signature
public enum ComponentContext implements Supplier<String> {
```

## Architecture & Concepts
ComponentContext is a type-safe enumeration that defines the operational context for an NPC's AI sensor components. Its primary architectural role is to enforce a constrained and explicit set of perspectives from which a sensor can evaluate the game world, eliminating the ambiguity and potential for error associated with using raw string identifiers.

This enum acts as a contract between the NPC asset definition system and the runtime AI engine. By representing contexts like **SensorSelf**, **SensorTarget**, and **SensorEntity** as distinct, immutable objects, the system gains compile-time verification and improved clarity in the AI behavior logic.

The implementation of the Supplier interface is a design choice for interoperability, allowing the enum to be seamlessly used in systems that require a string provider, such as logging, debugging, or certain serialization frameworks, without sacrificing the type safety of the core system. The static pre-cached **NotSelfEntitySensor** set is a performance optimization, providing immediate access to a common subset of contexts used in targeting logic.

### Lifecycle & Ownership
-   **Creation:** Enum instances are constructed by the Java Virtual Machine during class loading. They are compile-time constants and are not instantiated dynamically by application code.
-   **Scope:** Application-wide. An instance of each enum constant (e.g., SensorSelf) exists for the entire lifetime of the server process once the ComponentContext class is initialized.
-   **Destruction:** Instances are garbage collected by the JVM only when the application is shutting down and the class loader is unloaded. Their lifecycle is not managed by developers.

## Internal State & Concurrency
-   **State:** **Immutable**. Each enum constant is a singleton instance with a final *description* field. Its state cannot be modified after creation.
-   **Thread Safety:** **Inherently thread-safe**. As immutable singletons, instances of ComponentContext can be safely accessed, passed, and read from any thread without requiring locks or other synchronization primitives. The static **NotSelfEntitySensor** field is also safe for concurrent reads.

## API Surface
The public contract is minimal, focusing on providing the context's description and access to a pre-defined subset.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Returns the human-readable string description for the context. |
| NotSelfEntitySensor | EnumSet | O(1) | A static, pre-cached set containing contexts that are not self-referential. |

## Integration Patterns

### Standard Usage
ComponentContext is intended to be used as a configuration parameter when building or defining AI components. It provides a clear, self-documenting way to specify a sensor's focus.

```java
// Correctly configuring an AI sensor component using the enum
AIPresenceSensorBuilder sensorBuilder = new AIPresenceSensorBuilder();

// Set the sensor to focus on the NPC's current target
sensorBuilder.setContext(ComponentContext.SensorTarget);

// The AI system can now safely check the context
if (sensorBuilder.getContext() == ComponentContext.SensorTarget) {
    // Execute target-specific logic
}
```

### Anti-Patterns (Do NOT do this)
-   **String-Based Logic:** Avoid using the string value for conditional logic. This defeats the purpose of a type-safe enum and makes the code brittle to changes in the description text.
    ```java
    // BAD: Prone to typos and refactoring errors
    if (context.get().equals("target sensor")) { ... }

    // GOOD: Compile-time safety
    if (context == ComponentContext.SensorTarget) { ... }
    ```
-   **Manual Set Creation:** Do not manually recreate the **NotSelfEntitySensor** set in performance-critical code. Always use the provided static instance to avoid unnecessary object allocations.
    ```java
    // BAD: Inefficient in a loop or frequently called method
    EnumSet<ComponentContext> customSet = EnumSet.of(SensorTarget, SensorEntity);

    // GOOD: No allocation, direct access
    EnumSet<ComponentContext> efficientSet = ComponentContext.NotSelfEntitySensor;
    ```

## Data Pipeline
ComponentContext serves as a configuration value, typically originating from a static asset definition and flowing into the runtime AI system during NPC instantiation.

> Flow:
> NPC Asset File (JSON/YAML) -> Asset Deserializer -> **ComponentContext** -> AI Behavior Tree Builder -> Runtime AI Sensor Component

