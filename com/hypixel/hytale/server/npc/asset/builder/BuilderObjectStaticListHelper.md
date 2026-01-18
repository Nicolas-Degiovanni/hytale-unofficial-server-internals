---
description: Architectural reference for BuilderObjectStaticListHelper
---

# BuilderObjectStaticListHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderObjectStaticListHelper<T> extends BuilderObjectListHelper<T> {
```

## Architecture & Concepts
The BuilderObjectStaticListHelper is a specialized component within the server's asset-building framework, designed exclusively for deserializing and constructing lists of objects from static configuration sources, such as JSON files.

This class operates as a container and a delegate within a larger, hierarchical build process. When a configuration parser encounters a list or array of object definitions, it instantiates a BuilderObjectStaticListHelper to manage the construction of that entire list. The helper, in turn, creates and manages a series of child helpers (specifically, BuilderObjectStaticHelper instances) for each individual element within the list.

Its primary responsibility is to orchestrate the final build step for the list it represents. It iterates through its collection of child builders, invokes their respective build methods, and aggregates the results into a single, fully-realized Java List. The "Static" in its name is critical; it signifies that this helper and its children operate on data that is defined at load-time, not generated dynamically at runtime.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level parser or builder, typically when a JSON array is being processed. The parent builder that creates it is registered as its owner via the BuilderContext.
-   **Scope:** This object is short-lived and its scope is strictly confined to a single asset-building operation. It holds the intermediate representation of a list during parsing and is discarded after the final list is constructed.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the `staticBuild` method returns its result to the caller. It maintains no references or state that would cause it to persist beyond the build process.

## Internal State & Concurrency
-   **State:** The BuilderObjectStaticListHelper is **mutable**. Its core state is an internal list of `builders`, which is populated incrementally as a configuration file is parsed. This state is intended for write-once, read-once access; it is built up during parsing and consumed during the final `staticBuild` call. It does not cache the resulting built list.

-   **Thread Safety:** This class is **not thread-safe** and must not be accessed from multiple threads. It is designed to operate exclusively on the thread responsible for asset loading. The internal collection of builders is not synchronized, and concurrent access will lead to unpredictable behavior and data corruption.

## API Surface
The public contract is minimal, exposing only the final build operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| staticBuild(BuilderManager manager) | List<T> | O(N) | Orchestrates the construction of the final list. Iterates through N internal builders, invoking `staticBuild` on each. Returns a fully populated list or null if the source was empty. |

## Integration Patterns

### Standard Usage
This class is an internal component of the builder framework and is not intended for direct use by end-developers. The framework's BuilderManager orchestrates its use during asset loading. The following conceptual example illustrates its role.

```java
// Conceptual example of the framework's usage
BuilderManager manager = getBuilderManager();
SomeParentAssetBuilder parentBuilder = new SomeParentAssetBuilder();

// The parent builder would internally create and populate the helper
// while parsing a configuration source.
BuilderObjectStaticListHelper<NPCComponent> listHelper = parentBuilder.getComponentListHelper();

// During the final build phase, the manager invokes the helper
List<NPCComponent> components = listHelper.staticBuild(manager);

// The resulting list is then attached to the parent asset
parentAsset.setComponents(components);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new BuilderObjectStaticListHelper()`. It is tightly coupled to the lifecycle and context provided by the parent builder that must create it.
-   **State Re-use:** Do not call `staticBuild` more than once on the same instance. The helper is a single-use object designed for a one-way build process.
-   **Concurrent Modification:** Do not attempt to add builders to the helper from another thread while the main asset loading thread is processing. All operations must be serialized.

## Data Pipeline
The BuilderObjectStaticListHelper acts as a crucial aggregation step in the data flow from raw configuration to in-memory game objects.

> Flow:
> JSON Configuration File -> Framework Parser -> **BuilderObjectStaticListHelper** (aggregates child builders) -> `staticBuild` Invocation -> Final `List<T>` -> Game Asset Registry

