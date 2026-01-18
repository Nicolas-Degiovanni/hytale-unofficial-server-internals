---
description: Architectural reference for UIDisplayMode
---

# UIDisplayMode

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Utility

## Definition
```java
// Signature
public class UIDisplayMode implements Metadata {
```

## Architecture & Concepts
The UIDisplayMode class is an immutable, type-safe representation of a user interface display configuration. It functions as a piece of **Metadata** within the engine's schema and configuration system. Architecturally, it embodies the **Command Pattern**, where each static instance (NORMAL, COMPACT, HIDDEN) represents a pre-defined command that can be executed against a Schema object.

Its primary role is to decouple the declaration of a UI state from its application. A system can create or hold a reference to a UIDisplayMode instance without needing immediate access to the target Schema. When the configuration is to be applied, the instance's *modify* method is invoked, which then directly mutates the state of the provided Schema. This design ensures that all modifications to the UI display mode are centralized and consistent.

The use of a private constructor and public static final instances is a classic implementation of the **Type-Safe Enum Pattern** or **Flyweight Pattern**, preventing the creation of invalid or duplicate mode objects and promoting instance reuse for memory efficiency.

## Lifecycle & Ownership
- **Creation:** The three public instances, NORMAL, COMPACT, and HIDDEN, are instantiated exactly once by the JVM during class loading. Their lifecycle is not managed by any dependency injection framework or factory.
- **Scope:** As static final fields, these instances persist for the entire lifetime of the application. They are globally accessible and shared.
- **Destruction:** The instances are eligible for garbage collection only when the UIDisplayMode class is unloaded by the class loader, which typically occurs at JVM shutdown.

## Internal State & Concurrency
- **State:** The class is **immutable**. Its internal *mode* field is final and set only once via the private constructor. The public static instances cannot be reassigned.
- **Thread Safety:** UIDisplayMode is inherently **thread-safe**. Due to its immutability, multiple threads can safely read and share the static instances without any need for synchronization. The *modify* method operates on an externally provided Schema object; the thread safety of the overall operation is therefore dependent on the thread safety of the Schema instance it is modifying, not on UIDisplayMode itself.

## API Surface
The public contract consists of the three static instances and the single modification method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NORMAL | UIDisplayMode | O(1) | Static constant representing the standard UI layout. |
| COMPACT | UIDisplayMode | O(1) | Static constant representing a condensed UI layout. |
| HIDDEN | UIDisplayMode | O(1) | Static constant representing a hidden UI. |
| modify(Schema) | void | O(1) | Applies the instance's display mode to the target Schema. |

## Integration Patterns

### Standard Usage
The intended use is to retrieve one of the predefined static instances and pass it to a system responsible for processing Metadata. That system will then invoke the modify method.

```java
// Assume 'configurator' is a service that processes Metadata
// and 'schema' is the central configuration object.

// 1. Select the desired display mode
UIDisplayMode desiredMode = UIDisplayMode.COMPACT;

// 2. Apply the metadata to the schema
// The configurator would internally call desiredMode.modify(schema);
desiredMode.modify(schema);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create new instances via reflection. This violates the design contract and can lead to unpredictable behavior in systems that rely on reference equality (==) for checking the mode.
- **Null Schema:** Passing a null Schema object to the *modify* method will result in a runtime exception. The method expects a non-null, valid Schema instance to operate on.

## Data Pipeline
UIDisplayMode does not process a stream of data. Instead, it acts as a carrier for a specific configuration instruction that is applied to a central Schema object.

> Flow:
> System Logic Selects Mode -> **UIDisplayMode.INSTANCE** -> Schema Processor -> `instance.modify(schema)` -> Schema State Updated

