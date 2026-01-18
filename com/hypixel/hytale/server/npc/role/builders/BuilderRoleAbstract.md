---
description: Architectural reference for BuilderRoleAbstract
---

# BuilderRoleAbstract

**Package:** com.hypixel.hytale.server.npc.role.builders
**Type:** Transient

## Definition
```java
// Signature
public class BuilderRoleAbstract extends BuilderRole {
```

## Architecture & Concepts
The BuilderRoleAbstract class serves as a foundational component within the server-side Non-Player Character (NPC) role system. Its primary architectural purpose is to establish a contract for NPC builder roles that are conceptual or transitional, and thus should not be physically instantiated in the game world.

This class acts as a specialized base for other roles, enforcing the principle that certain builder behaviors, such as a planning or resource-gathering phase, do not correspond to a spawnable entity. By inheriting from BuilderRoleAbstract, a concrete role implementation automatically signals to the NPC management system that it is ineligible for spawning, simplifying spawn validation logic. It is a key element in modeling complex, multi-stage NPC behaviors.

### Lifecycle & Ownership
- **Creation:** Instances of classes derived from BuilderRoleAbstract are created by the NPC RoleFactory or a similar configuration system when an NPC's behavior state machine transitions into a non-spawnable task. It is not intended for direct instantiation.
- **Scope:** The lifetime of a BuilderRoleAbstract derivative is tied directly to the NPC entity that possesses it. It exists only as long as the NPC is in the corresponding behavioral state.
- **Destruction:** The object is eligible for garbage collection when the NPC's role is changed or the NPC itself is despawned. There is no explicit destruction method.

## Internal State & Concurrency
- **State:** This class introduces no new state and is therefore stateless on its own. Any state is inherited from the parent BuilderRole class. Its primary function is to provide a constant, immutable behavior.
- **Thread Safety:** The class is inherently thread-safe due to its lack of mutable state. The overridden isSpawnable method can be safely invoked from any thread without requiring synchronization.

## API Surface
The public contract is minimal, focusing exclusively on overriding parent behavior.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isSpawnable() | boolean | O(1) | Overrides the parent implementation to always return false. This is the core function of the class, preventing the associated NPC from being spawned. |

## Integration Patterns

### Standard Usage
This class is not used directly. It is intended to be extended to create specific, non-spawnable builder roles.

```java
// CORRECT: Extend the class to define a new role that cannot be spawned.
public class NpcPlanningRole extends BuilderRoleAbstract {
    // This role now inherits the isSpawnable = false contract.
    // Add logic specific to the NPC's planning phase here.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance of BuilderRoleAbstract directly. It provides no unique functionality and is meant purely as a base for inheritance.
- **Contradictory Overrides:** Do not create a subclass that extends BuilderRoleAbstract and then overrides isSpawnable to return true. This violates the class's design contract and indicates a fundamental misunderstanding of the role hierarchy. If a role should be spawnable, it must extend from a different base class, such as BuilderRole.

## Data Pipeline
BuilderRoleAbstract does not participate in a data processing pipeline. Instead, it acts as a gate in a decision-making flow related to NPC lifecycle management.

> **Flow:**
> NPC Spawn Request -> Spawn System queries NPC for its current Role -> Role.isSpawnable() is called -> **BuilderRoleAbstract.isSpawnable()** -> Returns **false** -> Spawn operation is cancelled by the system.

