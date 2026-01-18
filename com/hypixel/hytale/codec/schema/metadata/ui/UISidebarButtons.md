---
description: Architectural reference for UISidebarButtons
---

# UISidebarButtons

**Package:** com.hypixel.hytale.codec.schema.metadata.ui
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class UISidebarButtons implements Metadata {
```

## Architecture & Concepts
The UISidebarButtons class is a specialized component within the engine's Schema Configuration framework. It functions as a data-carrier and a command, encapsulating a specific piece of UI configuration—the set of buttons to be displayed in the main game sidebar.

By implementing the Metadata interface, this class participates in a Command Pattern. Each Metadata implementation represents a distinct, atomic modification to be applied to a central Schema object. This design decouples the *definition* of a configuration (e.g., loading button data from a file) from its *application* to the live configuration state. UISidebarButtons is responsible for holding an array of UIButton configurations and applying them when its `modify` method is invoked by a schema processor.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level configuration loader or UI definition parser. Typically, this occurs during the deserialization of asset files (e.g., JSON, HOCON) that define a specific UI screen or layout.
- **Scope:** Short-lived and transient. An instance of UISidebarButtons exists only for the duration of a single schema build or modification process. It is created, its `modify` method is called, and it is then immediately eligible for garbage collection.
- **Destruction:** Managed by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The class is **shallowly immutable**. Its internal state consists of a final reference to an array of UIButton objects, which is populated at construction time and cannot be changed thereafter. While the UIButton objects themselves may be mutable, this class performs no mutations on them.
- **Thread Safety:** The class itself is thread-safe. Its immutable nature ensures that multiple threads can read its state or call the `modify` method without risk of data corruption. **Warning:** The Schema object passed to the `modify` method is almost certainly *not* thread-safe. All modifications to a central Schema must be externally synchronized by the calling system, typically by processing all Metadata objects on a single thread during a specific engine initialization phase.

## API Surface
The public contract is minimal, focused entirely on the application of its encapsulated data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| modify(Schema schema) | void | O(1) | Applies the stored button configuration to the target Schema. Throws NullPointerException if the provided schema is null. |

## Integration Patterns

### Standard Usage
This class is not retrieved from a service registry. It is instantiated directly as part of a larger configuration process. The standard pattern involves creating an instance and immediately using it to modify a Schema object.

```java
// A Schema object is being built or modified
Schema targetSchema = getActiveSchema();

// Define the buttons that will form the sidebar
UIButton profileButton = new UIButton("Profile", ...);
UIButton settingsButton = new UIButton("Settings", ...);

// Create the Metadata command object
Metadata sidebarConfig = new UISidebarButtons(profileButton, settingsButton);

// Apply the configuration to the schema
sidebarConfig.modify(targetSchema);
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not hold references to UISidebarButtons instances for later re-use. These objects represent a point-in-time configuration. Caching them can lead to applying stale UI data if the underlying asset definitions are reloaded.
- **External Mutation:** Do not attempt to modify the internal button array via reflection. The object's immutability is critical to ensuring predictable and repeatable schema builds.

## Data Pipeline
UISidebarButtons acts as a transient carrier in the engine's configuration loading pipeline. It translates a declarative asset definition into an imperative modification on a live configuration object.

> Flow:
> UI Definition Asset (JSON) -> Asset Deserializer -> **UISidebarButtons Instance** -> Schema Processor -> `modify(schema)` -> Finalized Schema

