---
description: Architectural reference for ReputationGroupComponent
---

# ReputationGroupComponent

**Package:** com.hypixel.hytale.builtin.adventure.reputation
**Type:** Data Component

## Definition
```java
// Signature
public class ReputationGroupComponent implements Component<EntityStore> {
```

## Architecture & Concepts
The ReputationGroupComponent is a fundamental data component within Hytale's entity-component system. It serves as a simple, immutable tag that associates an entity with a specific reputation group. This component does not contain any logic itself; its sole purpose is to hold a unique identifier, the reputationGroupId.

This design allows various game systems, particularly the core ReputationSystem, to efficiently query and categorize entities. For example, when a player interacts with an NPC, the ReputationSystem can retrieve this component from the NPC entity to determine which faction's reputation should be adjusted. It is the primary link between a specific in-world entity and the abstract concept of a faction or group in the reputation database.

## Lifecycle & Ownership
- **Creation:** Instances are created and attached to entities by the EntityStore or a higher-level entity factory, typically during world generation or when an entity is spawned from a prefab. The static method getComponentType indicates it is registered with and managed by a central component registry, likely the ReputationPlugin.
- **Scope:** The lifecycle of a ReputationGroupComponent is strictly bound to the lifecycle of the entity to which it is attached. It exists only as long as its parent entity is active in the world.
- **Destruction:** The component is destroyed and its memory is reclaimed when the parent entity is removed from the EntityStore. This process is managed automatically by the engine's entity management system.

## Internal State & Concurrency
- **State:** This component is **immutable**. The reputationGroupId is a final field, set once during construction and cannot be changed thereafter. This makes the component a simple, predictable data container.
- **Thread Safety:** As an immutable object, ReputationGroupComponent is inherently thread-safe for read operations. However, all interactions with components should be managed through the entity's owning thread, typically the main server tick loop. Direct access from asynchronous tasks is strongly discouraged to maintain world state consistency.

## API Surface
The public contract is minimal, focusing on data retrieval and registration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getComponentType() | static ComponentType | O(1) | Retrieves the globally registered type for this component class. Essential for lookups. |
| getReputationGroupId() | String | O(1) | Returns the unique identifier for the reputation group this entity belongs to. |
| clone() | Component | O(1) | Creates a new instance with the same reputationGroupId. Used by the engine for entity duplication. |

## Integration Patterns

### Standard Usage
This component is not meant to be used in isolation. It should be retrieved from an existing entity to make logical decisions based on faction or group affiliation.

```java
// Example: A system checks an entity's faction before proceeding
Entity targetEntity = world.getEntity(targetId);
ReputationGroupComponent repGroup = targetEntity.getComponent(ReputationGroupComponent.getComponentType());

if (repGroup != null && "royal_guards".equals(repGroup.getReputationGroupId())) {
    // Logic specific to the Royal Guards faction
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ReputationGroupComponent()` for direct use. Components are meaningless unless they are attached to an entity via the `EntityStore` or a similar entity management context.
- **State Assumption:** Do not assume an entity will always have this component. Always perform a null check after retrieving it, as not all entities belong to a reputation group.

## Data Pipeline
This component acts as a data source for other systems. It does not process data itself.

> Flow:
> Player Interaction (e.g., kill, trade) -> Game Event Fired -> **ReputationSystem** -> Retrieves **ReputationGroupComponent** from target entity -> Uses `reputationGroupId` to look up reputation rules -> Updates Player Reputation State

