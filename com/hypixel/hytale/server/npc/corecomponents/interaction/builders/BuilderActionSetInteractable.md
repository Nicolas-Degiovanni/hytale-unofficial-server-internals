---
description: Architectural reference for BuilderActionSetInteractable
---

# BuilderActionSetInteractable

**Package:** com.hypixel.hytale.server.npc.corecomponents.interaction.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionSetInteractable extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionSetInteractable class is a configuration-to-object translator within the server's NPC behavior system. It serves a single, critical purpose: to parse a JSON definition and construct an executable ActionSetInteractable instance. This class is not part of the runtime game loop; instead, it operates exclusively during the server's asset loading phase.

It acts as a bridge between declarative data (NPC behavior files in JSON format) and imperative code (the runtime Action objects). By calling `requireInstructionType`, it enforces a strict architectural constraint, ensuring that this action can only be defined within an *Interaction* instruction block. This prevents designers from creating logically inconsistent NPC behaviors, such as modifying interaction state outside of a player interaction context.

The use of a BooleanHolder for the `setTo` field is a key design choice. It allows the interactable state to be defined not just as a static true or false, but as a dynamic value resolved at runtime from the ExecutionContext. This enables complex behaviors where an NPC's interactability might depend on game state, player inventory, or other contextual factors.

## Lifecycle & Ownership
- **Creation:** Instantiated reflectively by the NPC asset loading framework when it encounters a corresponding action type in an NPC's JSON behavior definition. It is never created directly by user code.
- **Scope:** Extremely short-lived. An instance of this builder exists only for the duration of parsing a single JSON object.
- **Destruction:** The instance becomes eligible for garbage collection immediately after the `build` method is called and the resulting ActionSetInteractable is passed to the parent instruction builder. It holds no persistent references and is not managed by any registry.

## Internal State & Concurrency
- **State:** Highly mutable. The primary purpose of this class is to accumulate state from a JSON object via the `readConfig` method. Its fields (`setTo`, `hint`, `showPrompt`) serve as temporary storage before being baked into the immutable ActionSetInteractable object.

- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The NPC asset loading pipeline is designed to be a single-threaded process. Concurrent calls to `readConfig` or `build` on the same instance will result in a corrupted and unpredictable state.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | Action | O(1) | Constructs the final ActionSetInteractable instance. This is the terminal operation for the builder. |
| readConfig(JsonElement) | BuilderActionSetInteractable | O(1) | Populates the builder's internal state from a JSON source. **WARNING:** Calling this multiple times will overwrite previous configuration. |
| getSetTo(BuilderSupport) | boolean | O(1) | Retrieves the configured interactable state. May involve a dynamic lookup in the ExecutionContext. |

## Integration Patterns

### Standard Usage
This class is used exclusively by the internal NPC asset loading system. A developer will never interact with it directly. The framework's usage pattern is as follows:

```java
// PSEUDOCODE: Representative of internal framework logic
// 1. A JSON object for this action is found
JsonElement actionJson = ...;

// 2. The builder is instantiated and configured
BuilderActionSetInteractable builder = new BuilderActionSetInteractable();
builder.readConfig(actionJson);

// 3. The final, immutable Action is built and stored
Action runtimeAction = builder.build(builderSupport);
instruction.addAction(runtimeAction);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never manually create an instance with `new BuilderActionSetInteractable()`. NPC behavior should be defined entirely within JSON asset files.
- **State Re-use:** Do not hold a reference to a builder after calling `build`. These objects are transient and designed to be discarded. Attempting to re-configure and re-build will lead to unpredictable side effects in the asset system.
- **Calling Build Before Read:** Invoking `build` on a fresh instance without first calling `readConfig` will produce an Action with default values, which will almost certainly result in incorrect game logic.

## Data Pipeline
The class functions as a specific step in the data transformation pipeline that turns static asset files into live game objects.

> Flow:
> NPC Behavior JSON File -> Server Asset Loader -> **BuilderActionSetInteractable.readConfig()** -> **BuilderActionSetInteractable.build()** -> ActionSetInteractable Instance -> Stored in NPC Behavior Tree -> Executed by Interaction Instruction at Runtime

