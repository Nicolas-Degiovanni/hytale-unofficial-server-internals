---
description: Architectural reference for BuilderActionTest
---

# BuilderActionTest

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderActionTest extends BuilderActionBase {
```

## Architecture & Concepts
The BuilderActionTest class is a component within the server's NPC asset pipeline. It serves as a concrete implementation of the Builder pattern, responsible for translating a declarative JSON configuration into a runtime-executable `ActionTest` object. This class is explicitly designed for debugging and testing the engine's attribute evaluation system and should **never** be used in production assets.

Its core architectural function is to decouple the static asset definition (the JSON file) from the dynamic, in-game behavior. This is achieved through a two-phase process:

1.  **Configuration Phase:** The `readConfig` method parses a `JsonElement`, populating a series of internal `...Holder` objects. These holders do not store raw values but rather representations of configured data, which may include static values, variables, or expressions.
2.  **Resolution Phase:** At runtime, the created `ActionTest` object queries this builder for its configured values via methods like `getBoolean(BuilderSupport)`. These methods delegate to the corresponding `...Holder` object, which uses the provided `ExecutionContext` to resolve the final, context-sensitive value.

This pattern allows a single NPC asset definition to produce varied behaviors depending on the game state (e.g., world time, player proximity, NPC health) without requiring complex, hard-coded logic.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderActionTest is created by the server's asset loading system when it encounters an action of this type within an NPC behavior definition file. It is not intended for manual instantiation.
-   **Scope:** The builder's lifetime is intrinsically linked to the `ActionTest` object it creates. The `build` method passes a reference of the builder (`this`) to the `ActionTest` constructor, establishing ownership. The builder persists as long as its corresponding `ActionTest` instance is active in the game world.
-   **Destruction:** The object is eligible for garbage collection once the `ActionTest` instance that holds a reference to it is destroyed. There is no manual cleanup or `dispose` method.

## Internal State & Concurrency
-   **State:** The internal state of BuilderActionTest is **mutable**. It consists of a collection of `...Holder` fields (e.g., `booleanHolder`, `stringHolder`). The `readConfig` method directly modifies this state. Once configured, the state is typically not changed again for the lifetime of the object.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be configured and used within the single-threaded context of the server's main game loop or a dedicated asset loading thread. The `readConfig` method performs unsynchronized writes to its internal fields. Concurrent calls to `readConfig` or calls to `get...` methods while `readConfig` is executing on another thread will lead to undefined behavior and state corruption.

**WARNING:** All interactions with an instance of this class must be externally synchronized if multithreaded access is unavoidable. The standard engine design, however, ensures this happens on a single thread.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | ActionTest | O(1) | Constructs and returns the final runtime `ActionTest` object. This is the terminal operation of the build process. |
| readConfig(JsonElement) | BuilderActionTest | O(N) | Parses the provided JSON data to configure the builder's internal state. N is the number of properties in the JSON. |
| getBoolean(BuilderSupport) | boolean | O(1) | Resolves the configured boolean value at runtime using the provided execution context. |
| getDouble(BuilderSupport) | double | O(1) | Resolves the configured double value at runtime using the provided execution context. |
| getFloat(BuilderSupport) | float | O(1) | Resolves the configured float value at runtime using the provided execution context. |
| getInt(BuilderSupport) | int | O(1) | Resolves the configured integer value at runtime using the provided execution context. |
| getString(BuilderSupport) | String | O(1) | Resolves the configured string value at runtime using the provided execution context. |
| getAsset(BuilderSupport) | String | O(1) | Resolves the configured asset path at runtime using the provided execution context. |

## Integration Patterns

### Standard Usage
The class is used by the NPC asset system. A developer would define the action in a JSON file, and the engine would handle the lifecycle. The resulting `ActionTest` object would then use its reference to the builder to retrieve values during its execution tick.

```java
// Engine-level code (conceptual)

// 1. The engine parses an NPC JSON file and instantiates the builder.
BuilderActionTest builder = new BuilderActionTest();

// 2. The relevant JSON block is passed to configure the builder.
builder.readConfig(npcJson.get("myTestAction"));

// 3. The builder creates the runtime action object.
BuilderSupport support = engine.getBuilderSupport();
ActionTest runtimeAction = builder.build(support);

// 4. Later, during gameplay, the action executes and resolves a value.
// This call happens inside the ActionTest.execute() method.
int resolvedValue = builder.getInt(support);
```

### Anti-Patterns (Do NOT do this)
-   **Production Use:** This class is explicitly for debugging. Its presence in a production asset file indicates a configuration error. The `getBuilderDescriptorState` method returns `Experimental` as a programmatic guard.
-   **Direct Instantiation:** Do not manually instantiate this class with `new BuilderActionTest()` and then call `build()` without first calling `readConfig()`. This will result in an `ActionTest` object with uninitialized, default values, leading to incorrect behavior.
-   **State Mutation After Build:** Do not call `readConfig` on a builder instance that has already been used to build an `ActionTest` object. This would mutate the shared state and cause unpredictable behavior in the existing runtime action.

## Data Pipeline
The flow of data begins with a static asset file and ends with a dynamically resolved value used by an NPC in the game world.

> Flow:
> NPC Behavior JSON File -> Server Asset Parser -> **BuilderActionTest.readConfig()** -> `ActionTest` Object Instantiation -> Game Engine Tick -> `ActionTest.execute()` -> **BuilderActionTest.get...()** -> Resolved Runtime Value

