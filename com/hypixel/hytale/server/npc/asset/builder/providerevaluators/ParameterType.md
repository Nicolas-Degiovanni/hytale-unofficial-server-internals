---
description: Architectural reference for ParameterType
---

# ParameterType

**Package:** com.hypixel.hytale.server.npc.asset.builder.providerevaluators
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum ParameterType implements Supplier<String> {
```

## Architecture & Concepts
ParameterType is a foundational enumeration that provides a compile-time, type-safe representation of primitive data types used within the server-side NPC asset building system. It is designed to eliminate the use of "magic strings" (e.g., "int", "string") when defining or evaluating parameters for NPC providers, thereby preventing common runtime errors and improving code clarity.

By implementing the Java `Supplier<String>` interface, this enum establishes a formal contract for retrieving the canonical string representation of each type. This pattern is crucial for systems that need to serialize or deserialize NPC configurations, ensuring consistency between the in-memory enum representation and its on-disk or network format. It serves as a schema definition primitive within the NPC asset pipeline.

## Lifecycle & Ownership
- **Creation:** Instances are created and initialized by the Java Virtual Machine (JVM) during the class loading phase. This process is automatic and occurs only once. Direct instantiation by developers is impossible.
- **Scope:** Application-wide. The enum constants DOUBLE, STRING, and INTEGER are effectively static final instances that persist for the entire lifetime of the server application.
- **Destruction:** The enum and its instances are unloaded when the application's class loader is garbage collected, which typically coincides with server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a private final string for its description. This state is set at compile time and cannot be modified during runtime.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, ParameterType can be safely accessed and shared across any number of threads without requiring locks or other synchronization mechanisms. It is a constant.

## API Surface
The primary API consists of the enum constants themselves.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | String | O(1) | Implements the Supplier contract, returning the canonical string representation of the type (e.g., "double"). |

## Integration Patterns

### Standard Usage
ParameterType is intended to be used for identity comparison in conditional logic, such as switch statements, or as a type-safe key in data structures.

```java
// Example of a provider evaluator using the enum for type checking
public void evaluate(ParameterType type, String rawValue) {
    switch (type) {
        case INTEGER:
            int intValue = Integer.parseInt(rawValue);
            // ... process integer
            break;
        case DOUBLE:
            double doubleValue = Double.parseDouble(rawValue);
            // ... process double
            break;
        case STRING:
            // ... process string
            break;
        default:
            throw new IllegalArgumentException("Unsupported parameter type: " + type);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **String-based Comparison:** Never use the string value for comparison. The primary benefit of the enum is type-safe identity checking.
  - **BAD:** `if (type.get().equals("int")) { ... }`
  - **GOOD:** `if (type == ParameterType.INTEGER) { ... }`
- **Extensibility via Inheritance:** Enums in Java are final and cannot be extended. Do not attempt to create subclasses to add new parameter types. If new types are needed, the enum itself must be modified.

## Data Pipeline
ParameterType acts as a metadata component for validating and interpreting data, rather than a processing stage itself. It is used by parsers and evaluators to understand the structure of incoming NPC asset data.

> Flow:
> NPC Asset File (e.g., JSON) -> Asset Deserializer -> **ParameterType** (Used for type validation) -> ProviderEvaluator -> In-Memory NPC Object

