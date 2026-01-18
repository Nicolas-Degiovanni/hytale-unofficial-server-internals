---
description: Architectural reference for UIDefaultCollapsedState
---

# UIDefaultCollapsedState

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Value Object

## Definition
```java
// Signature
public class UIDefaultCollapsedState implements Metadata {
```

## Architecture & Concepts
The UIDefaultCollapsedState class is a specific implementation of the Metadata interface, designed to represent a single, immutable configuration flag: whether a UI component should be rendered in a collapsed state by default.

Architecturally, it functions as a **Command Object**. An instance of UIDefaultCollapsedState encapsulates a request to modify a Schema object without knowing anything about the receiver of the request. The `modify` method executes this command, applying the stored boolean state to the target Schema.

This pattern decouples the declaration of a configuration detail from its application. High-level systems can construct collections of Metadata objects, including UIDefaultCollapsedState, and apply them transactionally to a Schema object during a build or processing phase. This makes the configuration system modular and extensible.

### Lifecycle & Ownership
-   **Creation:** The primary instance, UNCOLLAPSED, is a static final field instantiated by the JVM during class loading. The constructor is private, which strictly prohibits direct instantiation elsewhere. This enforces the use of the provided static constants, following a pattern similar to a typesafe enum or Flyweight.
-   **Scope:** The static UNCOLLAPSED instance is application-scoped. It persists for the entire lifetime of the JVM process.
-   **Destruction:** The object is garbage collected when its class is unloaded by the JVM, typically at application shutdown. There is no manual cleanup mechanism.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal `collapsedByDefault` field is final and set only at construction time. Once an instance exists, its state can never be altered.
-   **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that instances can be safely shared and passed between threads without any external synchronization. The `modify` method is also inherently safe as it only mutates its input parameter, the Schema object. Any concurrency concerns reside within the Schema class, not UIDefaultCollapsedState.

## API Surface
The public contract is minimal, focusing exclusively on the static constant and the command execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| UNCOLLAPSED | static UIDefaultCollapsedState | O(1) | A pre-defined, shared instance representing the uncollapsed state (false). |
| modify(Schema schema) | void | O(1) | Applies the internal collapsed state to the target Schema. This operation is idempotent. |

## Integration Patterns

### Standard Usage
This class is not meant to be used in isolation. It is designed to be passed to a higher-level system, such as a schema builder or processor, which will invoke the `modify` method at the appropriate time. The following example demonstrates the direct effect of the `modify` method for clarity.

```java
// This is a low-level operation, typically orchestrated by a SchemaBuilder.
Schema targetSchema = new Schema();

// The UNCOLLAPSED instance is a command to set the UI state to not collapsed.
UIDefaultCollapsedState.UNCOLLAPSED.modify(targetSchema);

// At this point, the targetSchema object has had its internal
// 'uiCollapsedByDefault' property set to false.
```

### Anti-Patterns (Do NOT do this)
-   **Reflection-based Instantiation:** Do not attempt to bypass the private constructor using reflection. The class contract relies on the provided static instances to ensure system-wide consistency.
-   **State Querying:** Do not attempt to read the internal state of this object. It is a command, not a data transfer object. Its sole purpose is to *act upon* a Schema, not to be queried for its value.

## Data Pipeline
As a configuration object, UIDefaultCollapsedState acts as a carrier of intent within the schema definition pipeline. It does not process streaming data but rather represents a static piece of the final configuration.

> Flow:
> Configuration Source (e.g., JSON asset) -> Deserializer -> **UIDefaultCollapsedState instance** -> SchemaBuilder.applyMetadata() -> Finalized Schema

