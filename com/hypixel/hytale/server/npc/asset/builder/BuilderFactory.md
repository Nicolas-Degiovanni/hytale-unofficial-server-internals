---
description: Architectural reference for BuilderFactory
---

# BuilderFactory<T>

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Factory / Registry

## Definition
```java
// Signature
public class BuilderFactory<T> implements SchemaConvertable<Void>, NamedSchema {
```

## Architecture & Concepts
The BuilderFactory is a cornerstone of the server's data-driven asset system, particularly for complex objects like NPCs. Its primary function is to act as a **polymorphic deserializer and factory** for configuration-driven object creation.

At its core, this class decouples the generic asset loading pipeline from the concrete implementations of various component builders. It maintains a registry mapping a string identifier to a specific `Builder` implementation. When parsing a configuration file, typically JSON, the factory inspects a designated field (the `typeTag`, e.g., "Type") to determine which concrete builder to instantiate. This allows developers to define new NPC behaviors, components, or other entities entirely through configuration and code, without modifying the core asset loading logic.

The implementation of `SchemaConvertable` and `NamedSchema` is critical. It signifies the factory's deep integration with the engine's configuration validation and tooling pipeline. A BuilderFactory can self-describe the valid JSON structure it expects, generating a formal schema. This schema is used for asset validation during builds and can power developer tools, such as auto-completion and in-editor documentation.

### Lifecycle & Ownership
- **Creation:** A BuilderFactory is instantiated programmatically during the server's bootstrap or module initialization phase. It is created once for each logical category of buildable objects (e.g., one factory for NPC behaviors, another for NPC appearance modifiers). After instantiation, it is configured via repeated calls to the `add` method.
- **Scope:** The object's lifecycle is tied to the asset loading context for its specific category. It is a long-lived object, designed to persist for the entire server session once its initial configuration is complete.
- **Destruction:** The factory is eligible for garbage collection when the server shuts down or its owning context is unloaded. It holds no native resources and does not require an explicit destruction method.

## Internal State & Concurrency
- **State:** The BuilderFactory is highly mutable during its initial configuration phase. Its primary internal state is the `buildersSuppliers` map, which stores the registered builder types. Once the application's bootstrap phase is complete, this map should be considered effectively immutable.
- **Thread Safety:** This class is **not thread-safe for mutation**. The `add` method is a check-then-act operation on a standard HashMap, making it vulnerable to race conditions if called from multiple threads.

    **WARNING:** All registration of builders via the `add` method **must** be performed from a single thread during application initialization. Read operations, such as `createBuilder`, are thread-safe *only if* no concurrent writes are occurring.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(name, builder) | BuilderFactory<T> | O(1) | Registers a new builder supplier under a given name. Throws IllegalArgumentException if the name is a duplicate. **Not thread-safe.** |
| createBuilder(config) | Builder<T> | O(1) | The primary factory method. Inspects the JsonElement, identifies the type via the `typeTag`, and returns a new instance of the corresponding builder. |
| createBuilder(name) | Builder<T> | O(1) | Creates a builder instance directly from its registered name, bypassing JSON inspection. |
| toSchema(context, isRoot) | Schema | O(N) | Generates a validation schema for all registered builders. Complexity is linear to the number of registered builders. |

## Integration Patterns

### Standard Usage
The factory is first initialized and configured at startup. Later, the asset loading system uses the configured instance to deserialize JSON objects into concrete game components.

```java
// 1. During server initialization (single-threaded context)
BuilderFactory<NPCBehavior> behaviorFactory = new BuilderFactory<>(NPCBehavior.class, "Type");
behaviorFactory.add("Wander", WanderBehavior.Builder::new);
behaviorFactory.add("FollowTarget", FollowTargetBehavior.Builder::new);
// ... register factory instance with an asset manager or service locator

// 2. During asset loading (can be multi-threaded)
JsonElement behaviorConfig = npcAssetJson.get("behavior");
Builder<NPCBehavior> builder = behaviorFactory.createBuilder(behaviorConfig);
NPCBehavior behavior = builder.build(context, behaviorConfig);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call the `add` method after the initialization phase, especially from a different thread than the one reading from the factory. This will lead to race conditions and unpredictable behavior.
- **Lazy Initialization:** Do not register builders on-demand when an asset is first loaded. The factory is designed to be fully populated at startup to ensure deterministic behavior and allow for complete schema generation.
- **Misconfigured Type Tag:** Relying on a `typeTag` that may not exist in the source JSON is a common error. If a configuration block might lack the type tag, you **must** provide a `defaultBuilder` during factory construction to handle this case gracefully.

## Data Pipeline
The BuilderFactory is a key transformation step in the asset-to-object pipeline. It translates declarative configuration data into executable builder logic.

> Flow:
> NPC Asset File (JSON) -> Gson Parser -> `JsonElement` -> **BuilderFactory.createBuilder(json)** -> Specific `Builder<T>` Instance -> `builder.build()` -> Instantiated Game Object (e.g., WanderBehavior)

