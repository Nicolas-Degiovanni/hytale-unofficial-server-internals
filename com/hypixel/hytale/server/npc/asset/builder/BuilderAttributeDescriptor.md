---
description: Architectural reference for BuilderAttributeDescriptor
---

# BuilderAttributeDescriptor

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient / Data Transfer Object

## Definition
```java
// Signature
public class BuilderAttributeDescriptor {
```

## Architecture & Concepts
The BuilderAttributeDescriptor is a metadata class that defines the schema for a single attribute within the server's NPC asset building system. It does not hold the data for an NPC instance; instead, it describes the *rules* for an attribute, such as its name, data type, validation constraints, and default value.

This class acts as a foundational component of a declarative, schema-driven system. A collection of BuilderAttributeDescriptor objects collectively forms the complete schema for a complex asset type, such as an NPC behavior or model. The main NPC asset builder consumes these descriptors to parse, validate, and instantiate final asset objects from raw data sources like JSON or other configuration files.

Its fluent, builder-style API allows for a clean and readable definition of these complex schemas directly in code.

## Lifecycle & Ownership
-   **Creation:** Instances are created directly via the constructor, typically during a static initialization phase of a parent Builder class (e.g., a specific NPC type's builder). The object is then configured using its own fluent API methods.

-   **Scope:** A BuilderAttributeDescriptor is a long-lived object. It persists for the lifetime of the asset schema it belongs to. It is instantiated once when the server defines its asset types and remains in memory to be reused for the validation and creation of every NPC asset of that type.

-   **Destruction:** The object is eligible for garbage collection only when its parent schema or builder definition is unloaded, which typically happens only on server shutdown or during a hot-reload of game assets.

## Internal State & Concurrency
-   **State:** The internal state is highly **mutable** during its initial configuration phase. Each call to a method like required or validator modifies the internal fields of the instance. After this setup phase, the object should be treated as **effectively immutable** by the rest of the system.

-   **Thread Safety:** This class is **not thread-safe**. Its methods directly mutate internal state without any synchronization mechanisms.

    **WARNING:** All configuration of a BuilderAttributeDescriptor instance must be completed on a single thread before it is registered with or used by the broader asset system. Concurrent modification will lead to a corrupted and unpredictable schema state.

## API Surface
The public API is designed as a fluent interface for defining an attribute's schema.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| required() | BuilderAttributeDescriptor | O(1) | Marks the attribute as mandatory. |
| optional(defaultValue) | BuilderAttributeDescriptor | O(1) | Marks the attribute as optional and sets a default value. |
| computable() | BuilderAttributeDescriptor | O(1) | Flags the attribute as a value that can be computed at runtime if not provided. |
| validator(validator) | BuilderAttributeDescriptor | O(1) | Attaches a custom Validator instance for complex validation logic. |
| setEnum(clazz) | BuilderAttributeDescriptor | O(N) | Configures the attribute to accept values from a given Enum, where N is the number of enum constants. |
| length(min, max) | BuilderAttributeDescriptor | O(1) | Sets the minimum and maximum size for array-like attributes. |

## Integration Patterns

### Standard Usage
A BuilderAttributeDescriptor is typically defined as a field within a larger Builder class and configured in a static or instance initializer. This creates a reusable schema for all assets created by that builder.

```java
// Inside a hypothetical NpcBehaviorBuilder class
// This defines a "speed" attribute for an NPC behavior.

private static final BuilderAttributeDescriptor SPEED_ATTRIBUTE =
    new BuilderAttributeDescriptor(
        "speed",
        "float",
        BUILDER_STATE,
        "Movement Speed",
        "The movement speed of the NPC in blocks per second."
    )
    .optional("1.0") // Default speed is 1.0
    .validator(new FloatRangeValidator(0.1, 10.0));
```

### Anti-Patterns (Do NOT do this)
-   **Post-Registration Modification:** Do not modify a BuilderAttributeDescriptor after it has been registered with the main asset system. The system assumes the schema is stable after initialization, and later changes can lead to inconsistent validation behavior.
-   **Cross-Thread Configuration:** Never configure a single BuilderAttributeDescriptor instance from multiple threads. The object is not thread-safe and its state will become corrupted.
-   **Reusing Instances:** Do not use the same BuilderAttributeDescriptor instance to define two different attributes. Each attribute in a schema must have its own unique descriptor object.

## Data Pipeline
This class is not a direct participant in a data processing pipeline. Instead, it serves as the *configuration* or *schema* that governs how the pipeline operates.

> Flow:
> 1. **Schema Definition:** Developer creates **BuilderAttributeDescriptor** instances to define an asset's contract.
> 2. **System Initialization:** Descriptors are registered with a master Builder service.
> 3. **Data Ingestion:** Raw data (e.g., JSON) for an NPC asset is loaded.
> 4. **Builder Execution:** The Builder service uses the registered **BuilderAttributeDescriptor** to validate fields, apply defaults, and parse values from the raw data.
> 5. **Asset Creation:** A fully validated and instantiated NPC asset object is produced.

