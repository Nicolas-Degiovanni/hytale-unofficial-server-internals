---
description: Architectural reference for BuilderMotionSequence
---

# BuilderMotionSequence

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Abstract Builder / Transient

## Definition
```java
// Signature
public abstract class BuilderMotionSequence<T extends Motion> extends BuilderMotionBase<T> {
```

## Architecture & Concepts
The BuilderMotionSequence is an abstract base class that serves as a foundational component in the NPC asset loading pipeline. It is not a concrete motion itself, but rather a factory responsible for parsing and validating a declarative JSON configuration that defines a sequence of other motions.

Architecturally, this class implements the **Composite** design pattern. It acts as a container node in a tree of motion definitions, holding a list of child motions (the `steps` field). This allows for the construction of complex, nested behaviors from simple, reusable JSON blocks.

Its primary role is to bridge the gap between static data (JSON files) and the server's live, in-memory object representation of an NPC's behavior. It is invoked exclusively during the server's boot or asset-reload phase.

Because it is an abstract class, developers must extend it to create builders for specific types of motion sequences. The base class provides the core logic for parsing the list of motions, handling looping, and managing activation behavior, while subclasses are expected to provide the logic for instantiating the final, concrete `Motion` sequence object.

### Lifecycle & Ownership
- **Creation:** Instances are created dynamically by the `BuilderManager` service during the recursive parsing of an NPC's JSON configuration file. It is never instantiated directly.
- **Scope:** The lifecycle of a BuilderMotionSequence instance is extremely short. It is created, used to process a specific JSON array (`Motions`), and then immediately becomes eligible for garbage collection. It is a transient object that does not persist into the active game state.
- **Destruction:** The object is destroyed by the garbage collector once the asset loading process for its corresponding configuration block is complete. Its state is transferred to a more permanent `Motion` object that is attached to the NPC.

## Internal State & Concurrency
- **State:** The class is highly mutable. Its internal fields, such as `steps`, `looped`, and `restartOnActivate`, are populated by the `readConfig` method. This state is intended to be built once and then used for validation before being discarded.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed for synchronous, single-threaded execution within the server's main asset loading routine. Concurrent calls to `readConfig` or modification of its internal list will result in race conditions and a corrupted, unpredictable server state.

## API Surface
The public API is focused on the two primary phases of the asset pipeline: configuration and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement data) | BuilderMotionSequence | O(N) | Parses the provided JSON, populating the internal state. N is the number of motions in the sequence. |
| validate(...) | boolean | O(N) | Recursively validates this builder and all child motion builders in its `steps` list. N is the number of motions. |
| isLooped() | boolean | O(1) | Returns true if the sequence is configured to repeat after completion. |
| isRestartOnActivate() | boolean | O(1) | Returns true if the sequence should reset to the first motion when the NPC is activated. |

## Integration Patterns

### Standard Usage
A developer does not interact with this class directly in code. Instead, they define its behavior declaratively in an NPC's JSON configuration file. The system then uses this builder internally.

A concrete implementation would be used by the system as follows:

```java
// This code is executed deep within the server's BuilderManager
// It is not typical user code.

// 1. A concrete builder is instantiated
ConcreteMotionSequenceBuilder builder = new ConcreteMotionSequenceBuilder();

// 2. The relevant JSON block is passed for parsing
builder.readConfig(npcJson.get("someMotionSequence"));

// 3. The builder and its children are validated
boolean isValid = builder.validate(configName, validationHelper, context, scope, errors);
if (!isValid) {
    // Handle validation failures
}

// 4. A final Motion object is created from the builder's state (not shown in base class)
Motion sequence = builder.build();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of a subclassed BuilderMotionSequence using `new`. The asset loading system's `BuilderManager` is responsible for the entire lifecycle. Bypassing it will lead to an unvalidated and disconnected component.
- **State Re-use:** Do not attempt to cache or re-use a builder instance. They are designed as single-use, transient objects for a specific parsing task.
- **Post-Config Mutation:** Do not modify the internal state (e.g., the `steps` list) after `readConfig` has been called. Doing so can bypass critical validation logic and lead to server instability.

## Data Pipeline
The BuilderMotionSequence is a key transformation step in the NPC asset pipeline. It converts structured text data into a validated object graph.

> Flow:
> NPC JSON File -> GSON Parser -> `BuilderManager` -> **BuilderMotionSequence.readConfig()** -> Recursive Validation of all child motions -> In-Memory `Motion` Object Graph

