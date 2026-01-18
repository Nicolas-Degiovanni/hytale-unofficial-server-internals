---
description: Architectural reference for BuilderObjectReferenceHelper
---

# BuilderObjectReferenceHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Helper

## Definition
```java
// Signature
public class BuilderObjectReferenceHelper<T> extends BuilderObjectHelper<T> {
```

## Architecture & Concepts
The BuilderObjectReferenceHelper is a fundamental component in the server's asset loading and object composition system. It acts as a generic, polymorphic factory for object instances of type T, where T is typically a component within the NPC or entity system (e.g., a behavior, a trigger, a condition).

Its primary architectural role is to decouple component definitions. Instead of forcing large, monolithic JSON files, it enables a component-based design where complex objects are assembled from smaller, reusable parts. This is achieved by supporting two distinct patterns for defining a component within a JSON configuration:

1.  **Inline Definition:** The component's entire configuration is defined directly within the parent JSON object. The helper instantiates a dedicated `Builder` to handle this inline data.
2.  **Reference Definition:** The configuration provides a string key that references a component defined elsewhere (e.g., in another file). The helper resolves this reference through the central `BuilderManager` to find the correct `Builder` at build time.

This dual-mode capability is the cornerstone of the NPC asset system's flexibility. Furthermore, the helper manages advanced features like **Modifiers** (`BuilderModifier`), which allow a referencing component to inject state or parameters into the referenced component. This creates a powerful system of scoped, parameterized instantiation, analogous to passing props to a component in a UI framework. It also handles the distinction between file-level references and internal, within-asset references.

## Lifecycle & Ownership
-   **Creation:** An instance of BuilderObjectReferenceHelper is created by a parent `Builder` during its `readConfig` phase. It is not a standalone service but a subordinate helper object, constructed to manage a specific field or property of its owner.
-   **Scope:** The lifecycle of a BuilderObjectReferenceHelper is strictly tied to its owning `Builder`. It persists from the moment the configuration is parsed until the owning `Builder` is destroyed, which typically occurs after the entire asset graph has been built.
-   **Destruction:** The object is eligible for garbage collection as soon as its parent `Builder` is no longer referenced. It holds no global state and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The class is highly stateful and mutable. Its internal fields (`builder`, `fileReference`, `referenceIndex`, `modifier`) are populated during the `readConfig` call. This state represents the parsed configuration for a single component and dictates the behavior of subsequent `build` and `validate` calls. It does not cache the final built object of type T, but caches the intermediate `Builder` or a reference to it.

-   **Thread Safety:** This class is **not thread-safe** and must only be used within the single-threaded context of the asset loading pipeline. Its methods mutate internal state without any synchronization mechanisms. The reliance on a shared, mutable `ExecutionContext` and its scope manipulation (`setScope`, `popComponentState`) further solidifies its single-threaded design. Concurrent access will lead to race conditions and unpredictable behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | T | O(N) | Constructs and returns the final object instance. Complexity is dependent on the target builder's build process. Throws exceptions on configuration errors. |
| validate(...) | boolean | O(N) | Performs load-time validation of the configuration without building the object. Essential for pre-loading and asset verification. |
| readConfig(...) | void | O(K) | Parses a `JsonElement` and populates the helper's internal state. This is the primary initialization method and must be called before `build` or `validate`. |
| getBuilder(...) | Builder<T> | O(1) | Retrieves the underlying `Builder` for the component, either from an internal field (for inline definitions) or by looking it up in the `BuilderManager`. |
| isPresent() | boolean | O(1) | Returns true if the helper represents either an inline definition or a valid reference. |

## Integration Patterns

### Standard Usage
The helper is designed to be used by another `Builder` to delegate the creation of a complex property. The parent builder instantiates the helper, passes it the relevant JSON data, and later calls `build` to retrieve the final object.

```java
// Inside a parent Builder's readConfig method:
this.myComponentHelper = new BuilderObjectReferenceHelper<>(MyComponent.class, this.getOwner());
this.myComponentHelper.readConfig(jsonObject.get("myComponent"), builderManager, ...);

// Inside the parent Builder's build method:
MyComponent component = this.myComponentHelper.build(builderSupport);
if (component != null) {
    // ... use the component
}
```

### Anti-Patterns (Do NOT do this)
-   **State Reuse:** Do not reuse a BuilderObjectReferenceHelper instance by calling `readConfig` on it multiple times. Its internal state is not designed to be reset. Create a new instance for each configuration block.
-   **Premature Build:** Do not call `build` or `validate` before `readConfig` has been successfully executed. This will result in `NullPointerException` or other runtime errors.
-   **External State Manipulation:** Do not modify the public fields of this class from an external context. All state should be managed internally via the `readConfig` method.
-   **Multi-threading:** Do not share an instance of this class across threads or invoke its methods concurrently. The entire asset building process is single-threaded.

## Data Pipeline
The BuilderObjectReferenceHelper is a key processing stage in the transformation of raw JSON configuration into a live game object.

> Flow:
> `JsonElement` -> `readConfig()` -> **BuilderObjectReferenceHelper** (State is populated: inline vs. reference) -> `build()` -> `BuilderManager` (Resolves reference if needed) -> Target `Builder<T>.build()` -> **Instance of T**

