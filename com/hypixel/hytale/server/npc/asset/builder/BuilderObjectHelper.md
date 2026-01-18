---
description: Architectural reference for BuilderObjectHelper
---

# BuilderObjectHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class BuilderObjectHelper<T> implements BuilderContext {
```

## Architecture & Concepts
The BuilderObjectHelper is an abstract base class that forms the foundational component of the server-side NPC asset loading system. It embodies a hierarchical, delegate-based builder pattern designed to transform raw JSON asset data into strongly-typed, in-memory Java objects.

Each concrete implementation of BuilderObjectHelper is an expert in parsing a specific fragment of an NPC's JSON definition. The system is structured as a tree of these helpers, where each helper can have an `owner` (another BuilderContext). This parent-child relationship allows for the resolution of complex, nested configurations and the propagation of contextual information, such as expression evaluation scopes, down the asset hierarchy.

This class acts as the critical bridge between the unstructured, text-based world of asset files and the structured, object-oriented domain of the running game server. Its primary responsibility is to encapsulate the logic for reading, validating, and ultimately constructing a single, well-defined piece of an NPC's configuration.

### Lifecycle & Ownership
The lifecycle of a BuilderObjectHelper is ephemeral and tightly controlled by the asset loading pipeline.

-   **Creation:** Instances are created dynamically by the BuilderManager during the traversal of an NPC's JSON asset tree. The manager selects and instantiates the appropriate concrete subclass based on the data type it needs to process.
-   **Scope:** The object's lifetime is strictly limited to the duration of a single asset build operation. It is a transient, single-use object designed to hold intermediate state for one specific build task.
-   **Destruction:** The entire tree of BuilderObjectHelper instances is eligible for garbage collection immediately after the root NPC object has been successfully constructed by the BuilderManager. No long-term references should ever be held to these objects.

## Internal State & Concurrency
-   **State:** Highly mutable. The primary purpose of a BuilderObjectHelper is to accumulate state from a JsonElement via the `readConfig` method. This state is held temporarily until the `build` method is invoked to construct the final object. It caches configuration data within its `builderParameters` field.
-   **Thread Safety:** **This class is not thread-safe.** The entire NPC asset building pipeline is designed to be a single-threaded process. All interactions with a BuilderObjectHelper instance and its subclasses must be confined to the server's main asset loading thread. Concurrent access will result in race conditions, inconsistent state, and unpredictable server behavior.

## API Surface
The public contract is designed for use by the BuilderManager and other components within the builder system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | T | Varies | **Abstract.** Constructs and returns the final object of type T. Subclasses implement the core materialization logic here. |
| validate(...) | boolean | Varies | **Abstract.** Performs load-time validation of the parsed data against game logic and constraints. |
| readConfig(...) | void | O(N) | Populates the helper's internal state from a given JsonElement. N is the size of the JSON fragment. |
| isPresent() | boolean | O(1) | **Abstract.** Returns true if the helper contains sufficient data to build a valid object. |
| getOwner() | BuilderContext | O(1) | Returns the parent context in the builder tree, allowing for upward traversal. |
| getClassType() | Class<?> | O(1) | Returns the target Class object that this helper is responsible for building. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. Instead, they create concrete subclasses to support new configurable NPC properties. The BuilderManager will then automatically discover and utilize these new helpers.

```java
// Example of a concrete implementation for a "Behavior" property
public class BehaviorBuilderHelper extends BuilderObjectHelper<NPCBehavior> {

    private String behaviorName;

    public BehaviorBuilderHelper(BuilderContext owner) {
        super(NPCBehavior.class, owner);
    }

    @Override
    public void readConfig(JsonElement data, ...) {
        // Logic to parse the behavior name from JSON
        this.behaviorName = data.getAsJsonObject().get("name").getAsString();
    }

    @Override
    public NPCBehavior build(BuilderSupport support) {
        // Logic to construct the final NPCBehavior object
        return new NPCBehavior(this.behaviorName);
    }

    // ... other required implementations
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never instantiate a subclass of BuilderObjectHelper directly. The BuilderManager is solely responsible for its creation and lifecycle management.
-   **State Reuse:** Do not attempt to cache or reuse a BuilderObjectHelper instance across multiple build operations. They are single-use and their internal state is not guaranteed to be valid after a build completes.
-   **Asynchronous Access:** Do not pass a helper instance to another thread or asynchronous task. All interactions must occur synchronously within the asset loading pipeline.

## Data Pipeline
This component is a central processing stage in the NPC asset loading data flow. It consumes raw JSON and produces a hydrated Java object.

> Flow:
> NPC Asset File (.json) -> GSON Parser -> JsonElement -> BuilderManager -> **BuilderObjectHelper** (`readConfig`) -> **BuilderObjectHelper** (`build`) -> In-Memory NPC Component (Type T)

