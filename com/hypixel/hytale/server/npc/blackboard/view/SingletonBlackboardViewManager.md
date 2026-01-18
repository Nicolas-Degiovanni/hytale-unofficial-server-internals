---
description: Architectural reference for SingletonBlackboardViewManager
---

# SingletonBlackboardViewManager

**Package:** com.hypixel.hytale.server.npc.blackboard.view
**Type:** Transient

## Definition
```java
// Signature
public class SingletonBlackboardViewManager<View extends IBlackboardView<View>> implements IBlackboardViewManager<View> {
```

## Architecture & Concepts
The SingletonBlackboardViewManager is a specialized implementation of the IBlackboardViewManager interface. Its primary architectural purpose is to enforce a single-instance pattern for a given Blackboard View, effectively treating a specific view as a global or world-level singleton within its managed scope.

Unlike other managers that might create, cache, or look up views based on entity ID, position, or chunk coordinates, this manager **ignores all contextual parameters**. Every request for a view, regardless of the arguments provided, returns the exact same, pre-configured view instance that was supplied during its construction.

This design is an optimization and simplification pattern used for AI data that is not entity-specific. It is ideal for representing global state, such as world properties (time of day, weather), faction reputations, or global event flags, where every NPC should see the identical data state. By always returning the same object, it avoids the overhead of lookups or per-entity allocations for shared, read-heavy data.

## Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor: `new SingletonBlackboardViewManager(view)`. The caller is responsible for creating and providing the managed `View` instance. This typically occurs during the initialization of a world or a specific AI system that requires a global blackboard.
-   **Scope:** The lifecycle of a SingletonBlackboardViewManager is bound to its owner. It is not a global engine singleton. It persists as long as the system that created it (e.g., a WorldAIManager) remains active.
-   **Destruction:** The owner is responsible for invoking the `cleanup()` or `onWorldRemoved()` methods. These calls are forwarded directly to the underlying `View` instance, allowing it to release its own resources. The manager itself has no state to clean up beyond its reference to the view and is garbage collected when its owner releases its reference.

## Internal State & Concurrency
-   **State:** The manager's internal state consists of a single `final` field, `view`, which holds the reference to the managed view instance. The manager itself is immutable after construction. However, the `View` object it holds is expected to be mutable, as it represents a dynamic view of the AI blackboard.
-   **Thread Safety:** This class contains no internal synchronization mechanisms. All methods are non-blocking field accesses or direct method delegations. Consequently, its thread safety is **entirely dependent on the thread safety of the wrapped View instance**. If the managed View is not thread-safe, concurrent calls to this manager from different AI agent threads can lead to race conditions and data corruption.

**WARNING:** Extreme caution is required when using this manager in a multi-threaded environment. The caller must ensure that the provided View is either thread-safe or that all access is externally synchronized.

## API Surface
The API contract is defined by the IBlackboardViewManager interface, but this implementation provides a constant-time, context-independent behavior for all methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(...) | View | O(1) | Ignores all parameters and returns the single, cached View instance. |
| getIfExists(long index) | View | O(1) | Returns the single, cached View instance. Fulfills the interface contract by always "existing". |
| cleanup() | void | O(N) | Delegates the cleanup call to the managed View. Complexity depends on the View's implementation. |
| onWorldRemoved() | void | O(N) | Delegates the world removal hook to the managed View. Complexity depends on the View's implementation. |
| forEachView(Consumer) | void | O(1) | Executes the provided consumer exactly once with the managed View as the argument. |
| clear() | void | O(1) | No-op. This method is intentionally empty and has no effect on the manager or the underlying View. |

## Integration Patterns

### Standard Usage
This manager is used when a single, shared view of AI data is required across all entities. The `View` is created first, then passed to the manager's constructor. The manager is then used by systems that need to access this global state.

```java
// 1. Create the shared view instance (e.g., a view for world state)
WorldStateView sharedWorldView = new WorldStateView(blackboard);

// 2. Create the manager to serve this single instance
IBlackboardViewManager<WorldStateView> manager = new SingletonBlackboardViewManager<>(sharedWorldView);

// 3. Any part of the AI system can now get the view.
// Note that the entity reference is ignored.
WorldStateView viewForNpc1 = manager.get(npc1.getRef(), blackboard, accessor);
WorldStateView viewForNpc2 = manager.get(npc2.getRef(), blackboard, accessor);

// viewForNpc1 and viewForNpc2 are the same object.
assert viewForNpc1 == viewForNpc2;
```

### Anti-Patterns (Do NOT do this)
-   **Using for Per-Entity State:** Do not use this manager for views that should contain entity-specific data (e.g., an NPC's current target or inventory). All entities will receive the same view object, leading to catastrophic state corruption. Use a different manager, like a CachingBlackboardViewManager, for per-entity views.
-   **Expecting `clear` to Reset State:** Calling `manager.clear()` does nothing. Do not rely on it to clear or nullify the underlying view. State management must be handled within the `View` object itself.
-   **Ignoring View Lifecycle:** Forgetting to call `cleanup()` or `onWorldRemoved()` on the manager will cause a resource leak in the underlying `View` if it holds references to world objects or other disposable resources.

## Data Pipeline
This component does not transform data; it acts as a constant-value provider in a request-response flow. It intercepts any request for a view and serves a pre-determined instance, short-circuiting any logic that would normally look up or create a new view.

> Flow:
> AI Behavior Tree -> Request for Blackboard View (with entity context) -> **SingletonBlackboardViewManager** -> Ignores context, returns cached global `View` instance -> AI Behavior Tree uses shared `View`

