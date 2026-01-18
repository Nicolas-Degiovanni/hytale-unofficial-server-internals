---
description: Architectural reference for BuilderContext
---

# BuilderContext

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface BuilderContext {
```

## Architecture & Concepts
The BuilderContext interface establishes a formal contract for any object that provides contextual information within a hierarchical building process. It is a foundational component of the server's NPC asset construction system. Its primary role is to enable any part of the asset creation logic to understand its position and lineage within a larger, nested build operation.

This is not a data-carrying object. Instead, it provides metadata *about* the build process itself. By creating a chain of ownership through the `getOwner` method, the system can construct a "breadcrumb" trail. This trail is indispensable for generating precise error messages, debugging, and logging, allowing developers to pinpoint failures within complex, deeply nested NPC definitions (e.g., "Error in behavior 'Attack'|state 'Enraged'|action 'SwingAxe'").

The interface is typically implemented by `Builder` classes themselves, allowing a builder to serve as the context for its children builders, forming a recursive, tree-like structure.

## Lifecycle & Ownership
As an interface, BuilderContext does not have its own lifecycle. The lifecycle is entirely dictated by the implementing class.

- **Creation:** An object implementing BuilderContext is created at the start of a specific, scoped build task. For example, when an NpcBuilder begins processing the "behaviors" section of an asset file, it might create a BehaviorSetBuilder which implements this interface.
- **Scope:** The scope of a BuilderContext is transient and is strictly tied to the duration of its corresponding build operation. It exists only as long as its owner, the builder, is active.
- **Destruction:** The object is eligible for garbage collection as soon as the build operation it was contextualizing completes and the final asset is produced. There is no explicit destruction or cleanup method defined in the contract.

## Internal State & Concurrency
The interface itself is stateless. All methods are either abstract or default implementations that delegate to other methods. State management is the sole responsibility of the implementing class.

- **State:** Implementors are expected to hold state, specifically a reference to their owner and a label for themselves. This state is typically immutable after the builder's initialization.
- **Thread Safety:** This interface is not inherently thread-safe. The default methods, such as `getBreadCrumbs`, create new objects (StringBuilder) and are safe for concurrent invocation. However, the thread safety of the underlying state (e.g., the owner reference) depends entirely on the implementation. It is assumed that a single asset build process occurs on a single thread, making concurrency concerns negligible in the standard use case.

**WARNING:** Concurrent modification of the ownership chain by multiple threads will lead to unpredictable behavior and is not supported.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getOwner() | BuilderContext | O(1) | Returns the parent context in the hierarchy. The root of the hierarchy will return null. |
| getLabel() | String | O(1) | Returns the human-readable name for the current context scope (e.g., "Behaviors", "State:Attacking"). |
| setCurrentStateName(String) | void | O(1) | Optional hook for more dynamic builders to update their context, for example when iterating a list of states. |
| getParent() | Builder<?> | O(N) | Traverses the ownership chain upwards to find the nearest ancestor that is a Builder instance. Complexity is proportional to the depth of the hierarchy. |
| getBreadCrumbs() | String | O(N) | Recursively builds a delimited string representing the full path from the root context to the current one. Highly valuable for logging. |

## Integration Patterns

### Standard Usage
The primary pattern is to retrieve the context from a parent builder and use it for logging or to pass it down to a child builder. The `getBreadCrumbs` method is the most common entry point for consumers of the interface.

```java
// A builder method receives a context from its caller
public void buildBehavior(BuilderContext context, BehaviorConfig config) {
    // Log the entry point using the context's path
    log.info("Building behavior at: " + context.getBreadCrumbs());

    try {
        // ... building logic ...
    } catch (BuildException e) {
        // Enrich the exception with precise location information
        throw new BuildException("Failed to build behavior at: " + context.getBreadCrumbs(), e);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Incomplete Chain:** An implementation must never have a broken ownership chain. If `getOwner` returns null prematurely, it effectively orphans a branch of the build tree, rendering `getBreadCrumbs` and error reporting useless.
- **Stateful Default Methods:** Do not override default methods like `getBreadCrumbs` with implementations that rely on or modify external state. The contract assumes these are pure, read-only operations.
- **Performance-Critical Loops:** Avoid calling `getBreadCrumbs` repeatedly inside a tight, performance-critical loop. The method involves recursion and string concatenation, which can create unnecessary GC pressure. Cache the result if needed within a local scope.

## Data Pipeline
BuilderContext does not participate in a data pipeline directly. Instead, it acts as a metadata provider that runs parallel to the main data transformation pipeline for asset creation.

> Flow:
> Raw Asset File (JSON) -> Parser -> **NpcBuilder (implements BuilderContext)** -> **BehaviorBuilder (implements BuilderContext)** -> Compiled NPC Asset
>
> During this flow, any builder can query its `BuilderContext` to understand its location and report errors with high fidelity.

