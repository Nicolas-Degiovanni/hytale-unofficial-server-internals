---
description: Architectural reference for BuilderActionResetInstructions
---

# BuilderActionResetInstructions

**Package:** com.hypixel.hytale.server.npc.corecomponents.utility.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionResetInstructions extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionResetInstructions class is a configuration-driven factory within the server-side NPC (Non-Player Character) behavior system. Its sole responsibility is to translate a static data definition from a JSON asset into a live, executable `Action` object.

This builder acts as a bridge between the declarative world of asset configuration and the imperative world of the server's game logic. Developers define an NPC's ability to reset its internal state machines or behavior trees in a JSON file. During server startup or asset loading, the engine instantiates this builder to parse that specific JSON block.

The core concept is the resolution of symbolic names. The JSON configuration specifies a list of human-readable instruction names. This builder uses the contextual `BuilderSupport` service to resolve these names into efficient integer-based identifiers (slots or indices) that the runtime execution engine uses. If no instructions are specified, it configures the resulting `Action` to reset *all* instructions for the NPC, providing a powerful "full state reset" mechanism.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the NPC asset parsing system when it encounters an action of this type within an NPC's behavior definition file. It is never created directly by game logic code.
- **Scope:** The lifecycle of a BuilderActionResetInstructions instance is extremely short and confined to the asset loading phase. It exists only to process a single JSON object, build the corresponding `Action`, and is then immediately eligible for garbage collection.
- **Destruction:** The instance is destroyed by the Java Garbage Collector once the `build` method has been called and the reference is dropped by the asset loader. It holds no persistent state beyond this process.

## Internal State & Concurrency
- **State:** This class is stateful but transient. The `readConfig` method populates the internal `instructions` field, making the object's state mutable during its brief lifecycle. This state is isolated to the specific configuration being parsed.

- **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed to be instantiated, configured, and used within the confines of a single-threaded asset loading pipeline. Concurrent calls to `readConfig` or `getInstructions` on a shared instance will result in a race condition and undefined behavior.

## API Surface
The public API is designed for use by the internal asset loading framework.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs the final `ActionResetInstructions` runtime object. |
| readConfig(JsonElement) | BuilderActionResetInstructions | O(N) | Parses the JSON configuration, populating internal state. N is the number of instruction names. |
| getInstructions(BuilderSupport) | int[] | O(M) | Resolves the stored instruction names into integer IDs. M is the number of instruction names. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by developers. It is invoked automatically by the engine's asset loading systems. The standard interaction pattern is declarative, through a JSON asset file.

The following example demonstrates the engine's internal usage pattern after parsing the corresponding JSON.

```java
// Engine-level code (conceptual)
// 1. Engine instantiates the builder for a specific JSON block.
BuilderActionResetInstructions builder = new BuilderActionResetInstructions();

// 2. Engine passes the JSON data to the builder.
builder.readConfig(jsonElementFromAsset);

// 3. Engine provides a context and builds the final runtime action.
BuilderSupport context = ...; // Provided by the NPC asset manager
ActionResetInstructions runtimeAction = (ActionResetInstructions) builder.build(context);

// 4. The builder is now discarded and the runtimeAction is added to the NPC.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually instantiate this class in game logic using `new BuilderActionResetInstructions()`. NPC behaviors must be defined in assets to ensure correctness and maintainability.
- **State Re-use:** Do not attempt to reuse a builder instance to parse a second JSON configuration. Each builder is designed for a single, one-shot parsing operation.
- **Pre-emptive Resolution:** Do not call `getInstructions` before `readConfig` has been successfully invoked. Doing so will result in an empty or invalid array of instruction indices.

## Data Pipeline
The primary function of this class is to act as a transformation step in the data pipeline that converts static assets into executable server logic.

> Flow:
> NPC Behavior JSON Asset -> Engine Asset Parser -> **BuilderActionResetInstructions** -> `ActionResetInstructions` Object -> NPC Behavior Executor<ctrl63>

