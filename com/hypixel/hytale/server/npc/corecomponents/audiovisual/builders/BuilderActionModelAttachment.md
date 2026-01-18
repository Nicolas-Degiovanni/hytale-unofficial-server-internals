---
description: Architectural reference for BuilderActionModelAttachment
---

# BuilderActionModelAttachment

**Package:** com.hypixel.hytale.server.npc.corecomponents.audiovisual.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionModelAttachment extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionModelAttachment class is a key component within the server-side NPC (Non-Player Character) asset definition framework. It functions as a data-driven configuration parser that translates a JSON definition into a concrete game action object.

Its primary role is to act as an intermediary during the asset loading phase. It does not perform the model attachment itself; instead, it follows the Builder pattern to configure and construct an immutable ActionModelAttachment object. This resulting object is a runtime representation of the "attach model" action, which is later executed by the NPC's action processing system.

This class is a specific implementation of the abstract BuilderActionBase, signifying its place within a larger system of configurable NPC actions. This design allows game designers to define complex NPC behaviors declaratively in JSON files, which are then parsed by a suite of specialized builders like this one.

## Lifecycle & Ownership
- **Creation:** Instantiated by a higher-level asset factory or parser when it encounters an action of type "ModelAttachment" within an NPC's JSON configuration file.
- **Scope:** The lifecycle of a BuilderActionModelAttachment instance is extremely short. It exists only for the duration of parsing a single JSON object. After its build method is called to produce the final ActionModelAttachment, the builder instance is no longer referenced and becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java garbage collector. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The readConfig method populates the internal slot and attachment fields based on the provided JSON data. The state is held within two StringHolder objects, which can resolve dynamic values at runtime.

- **Thread Safety:** This class is **Not Thread-Safe** and is intended for single-threaded use. It is designed to be used exclusively within the asset loading pipeline, which is a sequential, single-threaded process. Concurrent calls to readConfig would corrupt the internal state of the builder.

## API Surface
The public API is designed for a simple, two-step process: configure, then build.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionModelAttachment | O(1) | Constructs and returns the final ActionModelAttachment object. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionModelAttachment | O(1) | Parses the input JSON, validates required fields ("Slot", "Attachment"), and populates the internal state. Returns itself for chaining. |
| getSlot(BuilderSupport) | String | O(1) | Resolves and returns the configured slot name. |
| getAttachment(BuilderSupport) | String | O(1) | Resolves and returns the configured attachment name. |

## Integration Patterns

### Standard Usage
This builder is intended to be used by the asset loading system. A new instance is created for each JSON action block, configured, and then used to build the final action object.

```java
// Hypothetical asset loading context
JsonElement actionJson = parseNpcActionFromFile("...");
BuilderSupport support = getBuilderSupport();

// 1. Instantiate the builder
BuilderActionModelAttachment builder = new BuilderActionModelAttachment();

// 2. Configure from source data
builder.readConfig(actionJson);

// 3. Build the final, immutable action object
ActionModelAttachment action = builder.build(support);

// The 'builder' instance is now discarded.
// The 'action' object is stored for later execution.
```

### Anti-Patterns (Do NOT do this)
- **Instance Re-use:** Do not reuse a builder instance to parse multiple, different JSON configurations. The internal state is not reset between calls to readConfig, which will lead to incorrect or merged configurations.
- **Build Before Configure:** Calling build on a newly instantiated builder without first calling readConfig will produce an ActionModelAttachment with empty, uninitialized data. This will likely cause a NullPointerException or other runtime assertion failure when the action is executed.
- **Direct State Modification:** The internal fields slot and attachment are protected for a reason. Do not attempt to subclass and modify them directly; all configuration must flow through the readConfig method to ensure proper validation.

## Data Pipeline
The class operates as a transformation step in the data pipeline that converts static asset definitions into live game objects.

> Flow:
> NPC Definition (JSON File) -> Server Asset Parser -> **BuilderActionModelAttachment.readConfig()** -> **BuilderActionModelAttachment.build()** -> ActionModelAttachment (Runtime Object) -> NPC Action Execution System

