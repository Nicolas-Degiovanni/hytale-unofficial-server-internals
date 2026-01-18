---
description: Architectural reference for BuilderModifier
---

# BuilderModifier

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderModifier {
```

## Architecture & Concepts

The BuilderModifier is a critical component in the server-side NPC asset pipeline. It represents a declarative, data-driven set of overrides applied to a base NPC definition. It is not a long-lived service but rather a transient object that encapsulates a set of modifications parsed from an NPC's JSON asset file.

Architecturally, this class acts as a bridge between the static asset definition (JSON) and the dynamic NPC instantiation process. Its primary responsibility is to parse a `Modify` block within an asset, compile the specified expressions, and hold them in a ready-to-use format.

During NPC construction, the BuilderModifier is used to generate a `Scope` object. This `Scope` contains the evaluated results of the modification expressions, which effectively override the default parameters of the NPC being built. This system allows designers to create numerous variations of a base NPC type without duplicating the entire asset, by simply providing a small `Modify` block.

The class also handles special configuration keys for complex systems:
*   **_ExportStates:** Exposes specific internal animation or logic states for external systems to query.
*   **_InterfaceParameters:** Allows for context-sensitive modifications. An NPC can have different parameter overrides depending on the role it is fulfilling at spawn time, such as a quest-giver versus a standard enemy.
*   **_CombatConfig:** A direct reference to a separate balancing asset, decoupling combat tuning from the core NPC definition.
*   **_InteractionVars:** Provides typed variables for the NPC's interaction system.

## Lifecycle & Ownership

-   **Creation:** A BuilderModifier is instantiated exclusively through the static factory method `fromJSON`. This occurs deep within the server's asset loading system when a JSON file containing an NPC definition is parsed. A special singleton, `EmptyBuilderModifier.INSTANCE`, is used when no `Modify` block is present, following the Null Object pattern.
-   **Scope:** The object's lifetime is strictly bound to the asset parsing and NPC instantiation process. It is created, used to generate one or more `Scope` objects for newly created NPCs of a specific type, and is then immediately eligible for garbage collection. It holds no state that persists beyond this process.
-   **Destruction:** Cleanup is handled entirely by the Java Garbage Collector. There are no native resources or explicit `close` methods.

## Internal State & Concurrency

-   **State:** The BuilderModifier is **effectively immutable**. Its internal collections and fields are populated once in the constructor (which is called by the `fromJSON` factory) and are never modified again. It serves as a read-only container for the compiled modification data.
-   **Thread Safety:** Due to its immutable nature, this class is inherently **thread-safe**. Its methods can be safely called from multiple threads without external synchronization. The primary method, `createScope`, is designed to be safe as it produces a new, distinct `Scope` instance on each invocation, preventing any risk of shared mutable state between threads that might be building NPCs concurrently.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| fromJSON(jsonObject, params, helper, info) | BuilderModifier | O(N) | **[Static Factory]** Parses a JsonObject to create a new BuilderModifier. N is the number of keys in the `Modify` block. Throws SkipSentryException on malformed data. |
| createScope(support, params, globalScope) | Scope | O(K) | Creates a new Scope by evaluating all K expressions held by this modifier. This is the primary mechanism for applying the modifications. |
| applyComponentStateMap(support) | void | O(1) | Pushes this modifier's state mappings onto the target BuilderSupport. Used for temporary state context during a build. |
| popComponentStateMap(support) | void | O(1) | Pops the state mappings from the target BuilderSupport, restoring the previous state. |
| getCombatConfig() | String | O(1) | Returns the asset reference for the combat configuration, if one was defined. |
| getInteractionVars() | Map | O(1) | Returns the map of variables for the NPC interaction system. |
| isEmpty() | boolean | O(1) | Returns true if no modifications were parsed. The system often uses EmptyBuilderModifier instead. |

## Integration Patterns

### Standard Usage

This class is an internal component of the asset system and is not intended for direct use in gameplay logic. Its usage is orchestrated by higher-level asset builders. The conceptual flow is as follows.

```java
// This code is a conceptual representation of the internal asset loading process.

// 1. An asset loader provides the JSON object and necessary context.
JsonObject sourceJson = ...;
BuilderParameters builderParameters = ...;
StateMappingHelper stateHelper = ...;
ExtraInfo extraInfo = ...;

// 2. The factory method is called to parse the "Modify" block within the JSON.
BuilderModifier modifier = BuilderModifier.fromJSON(
    sourceJson,
    builderParameters,
    stateHelper,
    extraInfo
);

// 3. During NPC instantiation, the modifier is used to create a local scope.
// This scope contains the final, overridden values for the NPC's parameters.
Scope globalScope = getGlobalScope();
Scope npcInstanceScope = modifier.createScope(
    executionContext,
    builderParameters,
    globalScope
);

// 4. The new scope is used to retrieve configured values.
int health = npcInstanceScope.getInteger("health"); // Returns the modified value.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is `protected` for a reason. Never attempt to create an instance using `new BuilderModifier()`. The static `fromJSON` factory contains essential validation and processing logic that would be bypassed.
-   **Reusing Scopes:** The `Scope` object returned by `createScope` is specific to a single NPC instantiation context. Do not cache this `Scope` and reuse it for multiple NPCs, as it may contain instance-specific evaluations that are not applicable to others.
-   **Post-Creation Mutation:** A BuilderModifier is a snapshot of configuration at a point in time. Do not use reflection or other means to attempt to alter its internal maps after it has been created. This would violate its immutability guarantee and lead to unpredictable behavior in the NPC build process.

## Data Pipeline

The flow of data through this component is linear and unidirectional, transforming static configuration into an executable context.

> Flow:
> NPC Asset JSON File -> Gson Parser -> `JsonObject` -> **BuilderModifier.fromJSON** -> **BuilderModifier Instance** -> `createScope()` -> `Scope` Object -> NPC Instantiation Logic

