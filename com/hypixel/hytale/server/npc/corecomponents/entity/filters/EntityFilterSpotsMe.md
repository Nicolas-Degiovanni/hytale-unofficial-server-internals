---
description: Architectural reference for EntityFilterSpotsMe
---

# EntityFilterSpotsMe

**Package:** com.hypixel.hytale.server.npc.corecomponents.entity.filters
**Type:** Transient

## Definition
```java
// Signature
public class EntityFilterSpotsMe extends EntityFilterBase {
```

## Architecture & Concepts
The EntityFilterSpotsMe class is a server-side predicate component within the Non-Player Character (NPC) AI framework. Its primary function is to determine if a target entity is within the sensory perception of an NPC, specifically its field of view. This filter is a fundamental building block for creating complex AI behaviors such as aggro detection, guard patrols, and social interactions.

This class acts as a specialized filter that abstracts complex geometric calculations and world-state queries into a simple boolean evaluation. It is designed to be composed within higher-level AI constructs, such as Behavior Trees or Goal-Oriented Action Planners, where it serves as a condition to gate state transitions. For example, a "Patrol" behavior might transition to an "Attack" behavior only when an EntityFilterSpotsMe check passes for a hostile target.

Operationally, it queries the server's Entity-Component-System (ECS) for an NPC's **TransformComponent** (position) and **HeadRotation** (orientation) to establish a view frustum, which can be either a cone or a sector. It then checks if a target entity's position falls within this frustum and, optionally, performs a line-of-sight test to account for occluding geometry.

The filter includes a static **COST** value, indicating its relative performance impact. This allows the AI scheduler to potentially optimize behavior evaluation by prioritizing cheaper filters or limiting the execution frequency of expensive ones like this.

### Lifecycle & Ownership
- **Creation:** Instances are not created directly. They are exclusively constructed via a **BuilderEntityFilterSpotsMe** instance. This builder is typically configured from data files defining an NPC's archetype and instantiated by the server's NPC loading and initialization systems.
- **Scope:** The object's lifetime is bound to the specific AI behavior or role that owns it. It is not a global singleton and persists as long as its owning AI component is active in the world.
- **Destruction:** The object is eligible for garbage collection when the parent NPC, role, or behavior is unloaded or destroyed. There are no manual cleanup procedures required.

## Internal State & Concurrency
- **State:** This class is stateful but designed to be effectively immutable after construction. Configuration parameters like **viewAngle** and **testLineOfSight** are set once by the builder. However, it contains a mutable internal **view** Vector3d field. This vector is used as a temporary buffer for calculations to avoid heap allocations during frequent checks within the game loop.

- **Thread Safety:** **This class is not thread-safe.** The reuse of the internal **view** vector for performance reasons makes it inherently unsafe for concurrent access. If a single instance were to be used by multiple threads simultaneously, it would result in race conditions and unpredictable behavior. The design presumes that all AI updates for a given NPC, including filter evaluations, occur on a single, designated thread.

## API Surface
The public contract is focused on the evaluation of the filter against a target entity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matchesEntity(ref, targetRef, role, store) | boolean | O(1) | Evaluates if the target entity is spotted by the source entity. This is the core method. Complexity is constant time per check, involving several component lookups and vector math operations. |
| cost() | int | O(1) | Returns the static performance cost (400) associated with this filter. Used by AI scheduling systems. |

## Integration Patterns

### Standard Usage
This filter is not intended to be invoked directly by most systems. It is configured and then executed by an NPC's **Role** or an encapsulating AI behavior. The system retrieves the filter and uses it to test potential targets.

```java
// Hypothetical usage within an NPC's update logic
// Note: This is a conceptual example. You would not call this directly.

public void findTarget(Role npcRole, Store<EntityStore> store) {
    // The role would hold a list of filters configured for this NPC
    EntityFilterSpotsMe perceptionFilter = npcRole.getPerceptionFilter();
    
    for (Ref<EntityStore> potentialTarget : findNearbyEntities()) {
        boolean isSpotted = perceptionFilter.matchesEntity(
            npcRole.getSelfRef(), 
            potentialTarget, 
            npcRole, 
            store
        );

        if (isSpotted) {
            // Target is visible, initiate combat or other behavior
            npcRole.setTarget(potentialTarget);
            break;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is not public. You must use **BuilderEntityFilterSpotsMe** to create an instance. Attempting to bypass this will fail and violates the design contract.
- **State Mutation:** Do not attempt to modify the filter's properties via reflection after it has been constructed. The object is designed to be configured once.
- **Concurrent Execution:** **CRITICAL:** Never share a single **EntityFilterSpotsMe** instance across multiple threads. The internal state is not protected by locks and will become corrupted. Each NPC thread should operate on its own set of AI components.

## Data Pipeline
The data flow for a single `matchesEntity` evaluation is a sequence of state lookups and calculations.

> Flow:
> Entity References -> **EntityFilterSpotsMe.matchesEntity()**
> 1.  Fetch **TransformComponent** & **HeadRotation** for source NPC from Entity Store.
> 2.  Fetch **TransformComponent** for target entity from Entity Store.
> 3.  Perform geometric test (View Cone or View Sector) using **NPCPhysicsMath**.
> 4.  If test passes and **testLineOfSight** is true, delegate to **Role.PositionCache.hasInverseLineOfSight()** for a physics-based raycast check.
> 5.  Return final boolean result.

