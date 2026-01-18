---
description: Architectural reference for DebugSupport
---

# DebugSupport

**Package:** com.hypixel.hytale.server.npc.role.support
**Type:** Component

## Definition
```java
// Signature
public class DebugSupport {
```

## Architecture & Concepts

The DebugSupport class is a stateful component that encapsulates all diagnostic and debugging logic for a single Non-Player Character (NPC) Role. It is not a core gameplay system but rather a development and diagnostics tool that can be dynamically attached to an NPC to provide insight into its behavior.

Its primary architectural role is to act as a centralized, queryable configuration hub for an NPC's debug state. Other systems, such as the AI behavior tree, motion controllers, and server-side renderers, can interrogate this component to determine if they should execute their respective debug logic for the associated NPC.

A key design concept is the use of the `activate` method to "bake" the configuration from the `EnumSet` of RoleDebugFlags into primitive boolean fields. This is a performance optimization that allows for extremely fast checks in performance-critical code paths, such as the main game loop, avoiding the overhead of repeated `EnumSet.contains` lookups.

## Lifecycle & Ownership

-   **Creation:** An instance of DebugSupport is created by an NPC's Role, specifically during the role's construction phase via a BuilderRole. It is instantiated only when debugging is required for that specific NPC.
-   **Scope:** The lifecycle of a DebugSupport instance is strictly bound to the lifecycle of its parent NPCEntity's active Role. It persists only as long as the Role it supports is active. If the NPC changes roles, this instance is discarded and a new one may be created for the new role.
-   **Destruction:** The object is eligible for garbage collection as soon as its owning Role is replaced or the parent NPCEntity is destroyed. It does not manage any native resources and has no explicit destruction or cleanup method.

## Internal State & Concurrency

-   **State:** This class is highly mutable. Its core state is the `debugFlags` EnumSet, which can be changed at runtime. The boolean fields (e.g., `debugRoleSteering`, `traceFail`) are a mutable cache derived from this set. The string fields are designed for "consume-on-read" access, where a call to a `poll` method retrieves the value and simultaneously nullifies the internal state.

-   **Thread Safety:** **This class is not thread-safe.** All methods must be called from the main server thread. The internal state is not protected by locks or other synchronization mechanisms. Concurrent access, especially to the `poll` methods, will lead to race conditions and unpredictable behavior.

    **WARNING:** Do not store a reference to this object and access it from asynchronous tasks or other threads. All interactions must be synchronized with the server's main tick loop.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setDebugFlags(EnumSet) | void | O(N) | Sets the active debug flags and triggers `activate` to update internal state. N is the number of flags. |
| isDebugFlagSet(RoleDebugFlags) | boolean | O(1) | Checks if a specific debug flag is currently active. This is the primary query method. |
| pollDisplayCustomString() | String | O(1) | Retrieves the custom display string. **CRITICAL:** This is a destructive read; the internal string is set to null after this call. |
| pollDisplayPathfinderString() | String | O(1) | Retrieves the pathfinder display string. This is also a destructive read. |
| setLastFailingSensor(Sensor) | void | O(1) | Stores a reference to the last AI sensor that failed, for later inspection. |
| activate() | void | O(N) | Recalculates all internal boolean caches and re-creates the RoleDebugDisplay object based on the current `debugFlags` set. |

## Integration Patterns

### Standard Usage

The intended pattern is for other systems to retrieve the DebugSupport component from an NPC's Role and query it within the same game tick.

```java
// In an AI or rendering system...
NPCEntity npc = ...;
Role currentRole = npc.getRole();
DebugSupport debug = currentRole.getDebugSupport(); // May be null

if (debug != null && debug.isDebugFlagSet(RoleDebugFlags.SteeringRole)) {
    // Execute debug rendering for steering vectors
    renderSteeringVectors(npc);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new DebugSupport()`. The instance must be created by the NPC's Role system, which correctly injects the parent NPCEntity and initial flags. A manually created instance will be disconnected from the NPC and will not function.
-   **Caching Flags Externally:** Do not query a flag once and store the result in another system. The debug flags can be changed at any time by a developer command. Always query the DebugSupport instance directly to get the most current state.
-   **Ignoring Nulls:** The `getDebugSupport()` method on a Role may return null if debugging is not active. Always perform a null check before attempting to use the object.

## Data Pipeline

DebugSupport primarily functions as a control-flow mechanism rather than a data-processing pipeline. Its influence is based on external systems polling its state.

> Flow:
> Developer Command -> `setDebugFlags` -> **DebugSupport** (internal state change via `activate`) -> AI/Render System polls `isDebugFlagSet` -> Conditional Debug Logic Execution

