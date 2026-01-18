---
description: Architectural reference for BuilderActionRole
---

# BuilderActionRole

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionRole extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionRole class is a key component in the server's data-driven NPC asset pipeline. It functions as a deserializer and factory, responsible for translating a declarative JSON configuration into a concrete, executable game object.

Its primary role is to parse a JSON block that defines an NPC action to change its "Role". It validates the specified parameters, such as the target Role asset and optional state transitions, and stores them internally. The final `build` method then constructs an `ActionRole` instance, which is the runtime object that the NPC's behavior system can execute.

This class acts as a critical bridge between the static asset definition (the "what") and the dynamic runtime instruction (the "how"). It ensures that NPC behavior defined in external files is valid and correctly instantiated into the game world.

### Lifecycle & Ownership
-   **Creation:** Instantiated dynamically by the NPC asset loading system whenever it encounters an action of this type within an NPC's JSON definition file. A new instance is created for each distinct action block being parsed.
-   **Scope:** Extremely short-lived. A BuilderActionRole instance exists only for the duration of parsing a single JSON object. Its purpose is fulfilled once the `build` method is called and the resulting `ActionRole` object is returned.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the `build` method completes and its result is integrated into the parent NPC's behavior definition. It holds no persistent state or external resources requiring manual cleanup.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Its core purpose is to accumulate state from a JSON source via the `readConfig` method. The internal fields `role`, `changeAppearance`, and `state` are populated during this process. The state is transient and represents the configuration of a single action.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed for synchronous, single-threaded use within the asset loading pipeline. Concurrent calls to `readConfig` on the same instance will result in a corrupted, unpredictable state.

## API Surface
The public API is designed for a sequential, single-pass build process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(JsonElement) | Builder<Action> | O(1) | Deserializes the provided JSON, populating internal state. Validates inputs like asset existence. |
| build(BuilderSupport) | Action | O(1) | Constructs and returns a new `ActionRole` instance using the state populated by `readConfig`. |
| getRole(BuilderSupport) | String | O(1) | Retrieves the configured Role name. **Warning:** Must be called by the resulting `ActionRole`, not externally. |
| getChangeAppearance(BuilderSupport) | boolean | O(1) | Retrieves the configured appearance flag. **Warning:** Must be called by the resulting `ActionRole`. |
| getState(BuilderSupport) | String | O(1) | Retrieves the configured state name. **Warning:** Must be called by the resulting `ActionRole`. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is invoked automatically by the server's asset parsing framework. The typical lifecycle is managed internally by this framework.

```java
// Conceptual example of internal asset parser usage
JsonElement actionJson = ... // A JSON object for this action
BuilderSupport support = ... // The context for asset resolution

// The framework creates, configures, and builds in a single chain
ActionRole executableAction = (ActionRole) new BuilderActionRole()
    .readConfig(actionJson)
    .build(support);

// The executableAction is then added to an NPC's behavior tree
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** Never reuse a BuilderActionRole instance to parse more than one JSON configuration. The internal state is not reset between calls to `readConfig`, which will lead to data from previous configurations bleeding into new ones. Always create a new instance for each action.
-   **Building Before Reading:** Calling `build` before `readConfig` will produce an `ActionRole` with uninitialized data. This will cause runtime NullPointerExceptions or other assertion failures when the action is executed.
-   **External State Access:** Do not call the `getRole`, `getChangeAppearance`, or `getState` methods from outside the `ActionRole` object that this builder creates. These getters are part of the contract between the builder and its product, not for general-purpose querying.

## Data Pipeline
The BuilderActionRole is a transformation step in the NPC asset loading pipeline. It converts structured text data into an executable object.

> Flow:
> NPC Definition (`.json` file) -> Server Asset Parser -> `readConfig(JsonElement)` -> **BuilderActionRole (Internal State)** -> `build(BuilderSupport)` -> `ActionRole` Instance -> NPC Behavior Tree

