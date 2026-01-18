---
description: Architectural reference for BlockInteractionUtils
---

# BlockInteractionUtils

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Utility

## Definition
```java
// Signature
public class BlockInteractionUtils {
```

## Architecture & Concepts
BlockInteractionUtils is a stateless, server-side utility class that provides predicate functions for the core interaction system. Its primary role is to centralize and encapsulate specific game logic rules, decoupling them from the systems that process interactions like block breaking or placement.

This class acts as a decision-making component within the interaction pipeline. It directly queries the Entity Component System (ECS) to determine the context of an action. Specifically, it inspects an entity's components to check if it is a Player and, if so, what its current GameMode is. This allows other systems to easily differentiate between actions governed by standard survival rules (a "natural action") and those performed by players with special privileges, such as in a creative mode.

Its existence prevents the proliferation of redundant GameMode checks throughout the codebase, ensuring that the definition of a "natural action" is consistent across all interaction handlers.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a utility class containing only static methods, its methods are invoked directly on the class itself.
- **Scope:** The class and its static methods are available for the entire duration of the server's runtime, once loaded by the ClassLoader. It has no instance-level scope or lifecycle.
- **Destruction:** Not applicable. The class is unloaded when the server's ClassLoader is garbage collected, typically during a full server shutdown.

## Internal State & Concurrency
- **State:** BlockInteractionUtils is **stateless and immutable**. It contains no member fields and its methods' outputs are determined exclusively by their input arguments. It performs no caching.
- **Thread Safety:** This class is unconditionally **thread-safe**. Its stateless nature means there are no shared resources to protect. It is safe to call its methods from any thread, provided the arguments (such as the ComponentAccessor) are themselves used in a thread-safe manner for read operations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isNaturalAction(Ref, ComponentAccessor) | boolean | O(1) | Checks if an action is "natural". An action is natural if the source entity is not a Player, or if it is a Player in Adventure mode. Critically, returns true if the source entity Ref is null. |

## Integration Patterns

### Standard Usage
This class should be used by any system that needs to apply different logic based on whether an action is performed by a player under standard survival rules. It is invoked statically.

```java
// In a block processing system
void handleBlockBreak(Ref<EntityStore> sourceEntity, ComponentAccessor<EntityStore> accessor) {
    boolean isNatural = BlockInteractionUtils.isNaturalAction(sourceEntity, accessor);

    if (isNatural) {
        // Apply standard rules: check tools, drop items, etc.
    } else {
        // Apply creative/admin rules: break block instantly, no drops.
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new BlockInteractionUtils()`. This serves no purpose, wastes memory, and violates the intended static-only usage pattern.
- **Misinterpretation of Null:** The `isNaturalAction` method is designed to return `true` when the source entity `Ref` is `null`. This implies that actions without a clear origin (e.g., world generation, environmental effects) are treated as natural. Do not write code that assumes a `null` input will result in `false`, as this will lead to incorrect game logic.

## Data Pipeline
BlockInteractionUtils does not manage a data pipeline itself; rather, it serves as a conditional gate within a larger event processing pipeline.

> Flow:
> Server-Side Interaction Event (e.g., BlockBreakEvent) -> Event Handler -> **BlockInteractionUtils.isNaturalAction()** -> Conditional Game Logic (e.g., Drop Loot vs. No Loot)

