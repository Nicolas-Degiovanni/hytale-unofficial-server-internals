---
description: Architectural reference for the Builder interface, the core contract for NPC asset construction.
---

# Builder<T>

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface Builder<T> extends BuilderContext, SchemaConvertable<Void>, NamedSchema {
```

## Architecture & Concepts
The Builder interface is the fundamental contract for the server-side NPC asset construction pipeline. It defines a standardized process for converting a declarative JSON configuration into a concrete, in-memory game object of type T. This interface is the cornerstone of a highly extensible, schema-driven factory system managed by the BuilderManager.

Each implementation of Builder is a specialized factory responsible for a single type of asset component, such as an AI behavior, a physical attribute, or a visual effect. The system is designed to be data-driven; developers define NPC characteristics in JSON files, and the engine uses a corresponding set of Builder implementations to parse, validate, and instantiate the final NPC object graph at server load time.

Key architectural roles of a Builder are:
- **Deserialization:** Translating a specific `JsonElement` from an asset file into a structured object via the `readConfig` method.
- **Validation:** Enforcing semantic and logical correctness of the configuration through the `validate` method, preventing invalid game states before they occur.
- **Construction:** Assembling the final runtime object via the `build` method, using a provided `BuilderSupport` context to resolve dependencies.
- **Dependency Management:** Declaring both static and dynamic dependencies on other Builders, allowing the BuilderManager to construct assets in the correct topological order.

This interface acts as a powerful abstraction layer, decoupling the asset file format from the concrete Java classes that represent game logic.

## Lifecycle & Ownership
As an interface, Builder itself has no lifecycle. The following applies to its concrete implementations.

- **Creation:** Implementations are discovered and instantiated by the BuilderManager during the server's initial asset loading phase. This process is typically driven by reflection or a predefined registry, ensuring all available builders are known to the system.
- **Scope:** A Builder instance is a stateless, reusable singleton. It persists for the entire lifetime of the server process. It is designed to be invoked many times to build multiple objects of its designated category from different configurations.
- **Destruction:** Instances are destroyed and garbage collected only when the server shuts down and the BuilderManager itself is de-referenced.

## Internal State & Concurrency
- **State:** The Builder interface contract is fundamentally stateless. All necessary context for a build operation is passed in as method arguments (e.g., `BuilderSupport`, `ExecutionContext`). Implementations are **strongly expected** to be stateless and fully re-entrant. Storing per-build mutable state as instance fields is a severe anti-pattern that will cause unpredictable behavior.
- **Thread Safety:** The asset loading and building process is primarily single-threaded. Therefore, implementations are not required to be internally synchronized. However, adhering to the stateless design principle makes them inherently safe for potential future multi-threaded asset processing. Direct access from asynchronous game logic threads is not supported and will lead to race conditions.

**WARNING:** The methods `addDynamicDependency` and `clearDynamicDependencies` introduce mutability for advanced scenarios. Implementations that support this must exercise extreme caution, as this state is not inherently thread-safe and is managed by the single-threaded build orchestrator.

## API Surface
The public contract of the Builder interface focuses on the three main phases of asset creation: configuration, validation, and final construction.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readConfig(...) | void | O(N) | Parses the input `JsonElement`. N is the size of the JSON. This is the first step in the pipeline. |
| validate(...) | boolean | O(M) | Performs deep validation of the parsed configuration. M is the complexity of the validation rules. |
| build(support) | T | O(D) | Constructs and returns the final object. D is the number of resolved dependencies. Returns null if build fails. |
| getDependencies() | IntSet | O(1) | Returns a set of identifiers for other Builders this one depends on. Critical for the build orchestrator. |
| category() | Class<T> | O(1) | Returns the class token of the object type this Builder produces. Used for type-safe lookups. |
| isEnabled(context) | boolean | O(1) | A predicate to determine if this Builder should be active based on the current `ExecutionContext`. |

## Integration Patterns

### Standard Usage
A developer does not interact with a Builder instance directly. Instead, they define an asset in a JSON file. The engine's `BuilderManager` orchestrates the entire process.

```java
// PSEUDOCODE: Engine-level usage
// This logic is handled internally by the BuilderManager and is not for direct use.

// 1. Manager finds the right builder for a given JSON "type" field.
Builder<NPCBehavior> behaviorBuilder = builderManager.getBuilderForType("MeleeAttackBehavior");

// 2. Manager provides the JSON config to the builder.
behaviorBuilder.readConfig(context, jsonConfig, ...);

// 3. Manager validates the builder's state.
boolean isValid = behaviorBuilder.validate(...);
if (!isValid) {
    throw new AssetLoadException("Validation failed for MeleeAttackBehavior");
}

// 4. After all dependencies are built, the manager calls build.
// The BuilderSupport object provides access to already-built dependencies.
NPCBehavior behavior = behaviorBuilder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create a Builder using `new MyBuilder()`. The `BuilderManager` is the sole owner and manager of Builder instances. Direct creation bypasses dependency registration and lifecycle management.
- **Storing Per-Build State:** Do not add instance fields to a Builder implementation to store temporary data from `readConfig` for use in `build`. This breaks re-entrancy and will cause data corruption when building multiple assets. Pass all necessary data through the `BuilderSupport` context.
- **Ignoring Validation Results:** The boolean result of the `validate` method is critical. Proceeding to the `build` step after a validation failure will result in `NullPointerException` or other runtime exceptions.

## Data Pipeline
The Builder is a central component in the data transformation pipeline that converts static asset files into live server objects.

> Flow:
> NPC Asset JSON File -> `JsonElement` Parser -> **BuilderManager** (resolves which Builder to use) -> `Builder.readConfig` -> `Builder.validate` -> **Build Orchestrator** (resolves dependencies) -> `Builder.build` -> Instantiated Game Object (e.g., NPCBehavior)

