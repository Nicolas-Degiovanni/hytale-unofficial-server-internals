---
description: Architectural reference for BuilderComponent
---

# BuilderComponent<T>

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Factory

## Definition
```java
// Signature
public class BuilderComponent<T> extends BuilderBase<T> {
```

## Architecture & Concepts
The BuilderComponent is a fundamental part of the server-side NPC asset pipeline. It serves as a generic, configurable factory responsible for instantiating specific NPC behavior objects from JSON definitions. While its name includes "Component", it is more accurately a *builder for a component*, not the component itself.

The primary architectural pattern employed is **Delegation**. The BuilderComponent acts as a lightweight wrapper, delegating the complex logic of configuration parsing, validation, and object construction to an internal BuilderObjectReferenceHelper. This design promotes reusability; a single BuilderComponent class can be used to construct any type of object (e.g., an Action, a Motion, or a custom behavior) simply by parameterizing it with the target class type `T`.

Its role in the engine is to translate a declarative JSON object from an NPC asset file into a concrete, executable Java object that can be attached to an NPC entity at runtime. It is the bridge between static asset data and live game logic.

## Lifecycle & Ownership
The lifecycle of a BuilderComponent is strictly bound to the asset loading process. It is a transient, single-use object.

-   **Creation:** A BuilderComponent is instantiated by a higher-level factory or manager, such as a BuilderManager, during the parsing of an NPC definition file. The constructor is supplied with the specific `Class<T>` that this instance is responsible for building.
-   **Scope:** The object exists only for the duration of parsing and building one specific component from the configuration. It is created, configured via `readConfig`, validated, and then used a single time to produce an output object via `build`.
-   **Destruction:** After the `build` method returns the final constructed object, the BuilderComponent instance holds no further purpose and is eligible for garbage collection. It does not persist in the game state.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** during its short lifecycle. The `readConfig` method populates the internal BuilderObjectReferenceHelper with parsed data from the input JSON. Once configured, its state is effectively immutable for the subsequent `validate` and `build` calls.

-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. The entire asset loading pipeline is designed as a single-threaded, sequential process. All method calls (`readConfig`, `validate`, `build`) on a given instance must be performed by the same thread in the correct order. Concurrent access will lead to race conditions and an invalid final state.

## API Surface
The public API is designed to be invoked sequentially by the asset loading system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | T | O(N) | Terminal operation. Constructs and returns the final object of type T. Complexity depends on the referenced object's construction logic. Throws if not configured. |
| readConfig(JsonElement) | Builder<T> | O(M) | Populates the builder from a JSON definition. M is the size of the JSON data. This must be called before `build`. |
| validate(...) | boolean | O(V) | Validates the parsed configuration against engine rules. V is the complexity of the validation logic within the delegated helper. |
| canRequireFeature() | boolean | O(1) | A metadata check to determine if the component type (Action or Motion) can declare a dependency on a server feature. |
| toSchema(SchemaContext) | Schema | O(1) | Generates a JSON Schema definition for this component, used for tooling and offline validation. |

## Integration Patterns

### Standard Usage
Direct interaction with BuilderComponent is reserved for the NPC asset system. A developer defines the component in a JSON file, and the engine uses this class under the hood to realize it.

```java
// Conceptual engine-level usage
// This code is NOT for game logic developers.

// 1. Engine identifies a component definition for an Action
Class<Action> targetType = Action.class;
JsonElement componentJson = parseNpcFile().get("myActionComponent");

// 2. Engine creates and configures the builder
BuilderComponent<Action> builder = new BuilderComponent<>(targetType);
builder.readConfig(componentJson);

// 3. Engine validates the builder
boolean isValid = builder.validate(...);
if (!isValid) {
    throw new AssetLoadException("Validation failed for myActionComponent");
}

// 4. Engine builds the final object
BuilderSupport support = ...; // Provide engine context
Action finalAction = builder.build(support);

// 5. The finalAction is now ready to be used by an NPC
```

### Anti-Patterns (Do NOT do this)
-   **Reusing Instances:** A BuilderComponent is single-use. Do not attempt to call `readConfig` on an instance that has already been configured. This will result in corrupt or unpredictable state.
-   **Incorrect Lifecycle Invocation:** Calling `build` before `readConfig` has been successfully executed will result in a runtime exception, as the internal state is uninitialized.
-   **Caching Instances:** Do not cache and reuse BuilderComponent instances. They are lightweight and designed to be created and discarded for each component in a file.

## Data Pipeline
The BuilderComponent is a critical stage in the transformation of static data into a runtime object.

> Flow:
> NPC JSON File -> Json Parser -> **BuilderComponent.readConfig** -> Validation Engine -> **BuilderComponent.validate** -> Build-time Resolver -> **BuilderComponent.build** -> Instantiated Game Object (e.g., Action) -> NPC Runtime State

