---
description: Architectural reference for EntityFilterBase
---

# EntityFilterBase

**Package:** com.hypixel.hytale.server.npc.corecomponents
**Type:** Component Base Class

## Definition
```java
// Signature
public abstract class EntityFilterBase extends AnnotatedComponentBase implements IEntityFilter {
```

## Architecture & Concepts
EntityFilterBase is an abstract base class that forms the foundation of the server's entity selection and targeting system for Non-Player Characters (NPCs). It is a core component within the server's AI and behavior architecture.

The primary architectural purpose of this class is to establish a standardized, declarative contract for filtering collections of game entities. By subclassing EntityFilterBase, developers can create reusable and composable filtering logic. For example, specific implementations could filter for hostile entities, entities within a certain health threshold, or entities belonging to a specific faction.

This pattern decouples high-level AI behaviors (e.g., "find a target") from the low-level implementation of *how* a valid target is defined. This allows AI designers to mix and match filters in entity configuration files without altering core AI logic code. The inheritance from AnnotatedComponentBase suggests that filter implementations are discovered and configured by the engine using Java annotations, further promoting a data-driven design.

### Lifecycle & Ownership
-   **Creation:** Concrete implementations of EntityFilterBase are not instantiated directly using the *new* keyword. They are instantiated by the server's component registry or AI behavior system during the loading of an NPC's configuration. The engine is the sole owner and manager of filter instances.
-   **Scope:** The lifetime of a filter instance is bound to the AI behavior or component that references it. It persists as long as its parent component is active in the game world.
-   **Destruction:** Instances are marked for garbage collection when the owning NPC or its associated AI behavior is unloaded or destroyed.

## Internal State & Concurrency
-   **State:** EntityFilterBase is inherently stateless. Subclasses should strive to remain stateless to ensure predictable and deterministic AI behavior. If configuration is required (e.g., a filter for entities within a specific radius), these parameters should be injected during creation and treated as immutable for the lifetime of the object.
-   **Thread Safety:** **WARNING:** Filter logic is executed within the server's main entity processing loop, which may be multi-threaded to improve performance. All subclass implementations **must be thread-safe**. Avoid storing mutable instance variables that are modified during the filtering process, as this will lead to severe race conditions. Filters should be designed as pure functions where possible.

## API Surface
The public contract is defined by the IEntityFilter interface. The primary method is `test`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Entity) | boolean | Varies | (Inherited from IEntityFilter) Evaluates a single entity against the filter's criteria. Returns true if the entity passes the filter, false otherwise. This is the core method that all subclasses must implement. |

## Integration Patterns

### Standard Usage
Developers should extend EntityFilterBase to create custom filtering logic. The engine then uses these implementations, often specified in data files, to drive NPC behavior.

```java
// 1. A developer defines a concrete filter class.
// This filter identifies entities that are players and are not in creative mode.
public class NonCreativePlayerFilter extends EntityFilterBase {

    @Override
    public boolean test(Entity candidate) {
        if (!(candidate instanceof Player)) {
            return false;
        }
        Player player = (Player) candidate;
        return player.getGameMode() != GameMode.CREATIVE;
    }
}

// 2. The engine's AI system uses an instance of the filter.
// This code is representative of internal engine logic and is not
// typically written by a game developer.
IEntityFilter filter = componentRegistry.getFilter(NonCreativePlayerFilter.class);
List<Entity> nearbyEntities = world.getEntitiesInRadius(npc.getPosition(), 32);

List<Entity> validTargets = nearbyEntities.stream()
    .filter(filter::test)
    .collect(Collectors.toList());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new MyCustomFilter()`. The component system manages the lifecycle of filters. Attempting to manage them manually will bypass engine optimizations and can lead to memory leaks or incorrect behavior.
-   **Stateful Filtering:** Do not implement filters that change their internal state during a call to `test`. This breaks the assumption of determinism and can cause unpredictable AI oscillations and debugging nightmares in a concurrent environment.
-   **Heavyweight Operations:** The `test` method is in a hot path and will be called frequently. Avoid any blocking or computationally expensive operations like network requests, file system access, or complex pathfinding calculations within the filter itself. Pre-calculate and cache any required data.

## Data Pipeline
EntityFilterBase serves as a critical processing stage in the AI's target selection data pipeline.

> Flow:
> AI Behavior Tree Tick -> World Query (e.g., get entities in radius) -> Stream of Entities -> **EntityFilterBase.test(entity)** -> Filtered Collection of Entities -> AI Action (e.g., Attack, Follow)

