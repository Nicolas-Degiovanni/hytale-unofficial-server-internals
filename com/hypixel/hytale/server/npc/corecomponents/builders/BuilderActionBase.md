---
description: Architectural reference for BuilderActionBase
---

# BuilderActionBase

**Package:** com.hypixel.hytale.server.npc.corecomponents.builders
**Type:** Utility

## Definition
```java
// Signature
public abstract class BuilderActionBase extends BuilderBase<Action> {
```

## Architecture & Concepts
BuilderActionBase is an abstract foundational class within the server-side NPC (Non-Player Character) behavior system. It serves as the common template for all builders responsible for constructing concrete **Action** instances from configuration data.

In Hytale's architecture, NPC behaviors are defined declaratively, typically in JSON asset files. An **Action** represents a single, executable instruction within an NPC's behavior tree, such as moving to a location, playing an animation, or engaging a target. This class, and its concrete subclasses, form the critical bridge between the static JSON asset definitions and the live, executable **Action** objects used by the NPC instruction processor at runtime.

It embodies a Factory Pattern, where each subclass is responsible for understanding a specific type of action configuration and producing a fully initialized object ready for execution. This class specifically handles common configuration properties shared across most, if not all, actions, such as the "Once" flag.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of BuilderActionBase are not instantiated directly by game logic. They are discovered and instantiated by the NPC asset loading system during server initialization, likely through reflection or a service registry that maps action types to their corresponding builder classes.
- **Scope:** A single instance of each builder subclass persists for the entire server session. They are long-lived, reusable factories.
- **Destruction:** The builder instances are destroyed and garbage collected only when the server shuts down or a hot-reload of the asset system is triggered.

## Internal State & Concurrency
- **State:** This class contains a mutable state field, **once**. This field is populated during the invocation of the readCommonConfig method. The state is transient and relevant only for the duration of a single build operation. The builder object itself is stateful during the build but should be considered stateless between build operations.

- **Thread Safety:** This class is **not thread-safe**. The readCommonConfig method modifies the internal **once** field. The design presumes that the asset loading and NPC instantiation pipeline is a single-threaded process. Using the same builder instance from multiple threads concurrently to build different actions will result in a race condition and lead to unpredictable behavior in the resulting **Action** objects.

## API Surface
The public contract is designed for use by the NPC asset system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readCommonConfig(JsonElement data) | Builder<Action> | O(N) | Parses common configuration keys from the input JSON. N is the number of keys. Modifies internal state. |
| category() | Class<Action> | O(1) | Returns the Action.class token, identifying the type of object this builder family produces. |
| isEnabled(ExecutionContext context) | boolean | O(1) | A default implementation indicating that actions are always enabled. Subclasses may override for conditional logic. |
| isOnce() | boolean | O(1) | Returns true if the action being built should only be executed a single time. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers. It is a framework class to be extended. A developer defines a new NPC action by creating a new class that extends BuilderActionBase and implementing the abstract methods to produce a custom Action. The system then uses it automatically.

```java
// Example of a concrete implementation (system-level view)

// 1. A custom builder is defined
public class BuilderWalkToAction extends BuilderActionBase {
    // ... custom logic to read target coordinates
    @Override
    public Action build(BuildContext context) {
        // Creates a WalkToAction using parsed state, including the 'once' flag
        return new WalkToAction(this.target, this.isOnce());
    }
}

// 2. The system uses it during asset loading
BuilderRegistry registry = server.getNpcBuilderRegistry();
BuilderActionBase builder = registry.getBuilderFor("WalkTo"); // Finds BuilderWalkToAction
builder.readCommonConfig(actionJsonData);
Action newAction = builder.build(buildContext);
```

### Anti-Patterns (Do NOT do this)
- **State Bleed:** Do not read the state of a builder (e.g., by calling isOnce) outside of the immediate build process. The builder's state is only guaranteed to be valid during the parsing and building sequence for a single action.
- **Manual Management:** Do not instantiate or manage builder lifecycles manually. The asset system is the sole owner of these objects. Register your custom builder with the system and let it handle creation.

## Data Pipeline
This class is a key processing stage in the NPC asset-to-entity pipeline. It transforms declarative data into executable logic.

> Flow:
> NPC_Definition.json -> Server Asset Loader -> Gson Parser -> JsonElement -> **BuilderActionBase.readCommonConfig** -> Configured Action Instance -> NPC Behavior Controller

