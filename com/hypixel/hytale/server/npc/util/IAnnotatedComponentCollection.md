---
description: Architectural reference for IAnnotatedComponentCollection
---

# IAnnotatedComponentCollection

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IAnnotatedComponentCollection extends IAnnotatedComponent {
```

## Architecture & Concepts
The IAnnotatedComponentCollection interface is a foundational element of the server-side Non-Player Character (NPC) component system. It embodies the **Composite Design Pattern**, a structural pattern that allows clients to treat individual objects and compositions of objects uniformly.

By extending IAnnotatedComponent, this interface allows a collection of components to be treated as a single component. This is critical for creating hierarchical and modular NPC behaviors. For example, a high-level AI component like a CombatBehavior component might itself be an IAnnotatedComponentCollection containing smaller, more granular components such as AttackComponent, DefendComponent, and PathfindingComponent.

This design decouples systems that process components from the complexity of whether they are dealing with a single leaf-node component or a complex branch of components.

## Lifecycle & Ownership
As an interface, IAnnotatedComponentCollection does not have a concrete lifecycle. The following applies to its concrete implementations.

- **Creation:** Implementations are typically instantiated by NPC factories or assemblers when an NPC is spawned. They are rarely, if ever, created directly in game logic. The collection is populated with its child components during this assembly phase.
- **Scope:** The lifetime of a component collection is strictly tied to its parent entity, usually an NPC. It exists as long as the NPC is active in the world.
- **Destruction:** The collection and its contained components are marked for garbage collection when the parent NPC is despawned or destroyed. There is no manual destruction method defined in the contract.

## Internal State & Concurrency
- **State:** The contract implies mutable state. Implementations will hold a collection of IAnnotatedComponent instances. The size and contents of this collection may or may not be modifiable after initial creation, depending on the specific implementation.
- **Thread Safety:** This interface makes no guarantees about thread safety. Implementations are **not expected to be thread-safe**. All interactions with component collections and their children should be performed on the main server game thread to prevent race conditions and state corruption. Off-thread systems, such as asynchronous pathfinding, must post results back to the main thread before modifying component state.

## API Surface
The public contract is minimal, focusing on enumeration and access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| componentCount() | int | O(1) | Returns the number of components in the collection. |
| getComponent(int) | IAnnotatedComponent | O(1) | Retrieves a component by its index. **Warning:** Can return null. Callers must perform null checks. |

## Integration Patterns

### Standard Usage
The primary pattern is to iterate over the collection to apply logic to each component, or to use type-checking to find a specific child component within the group.

```java
// A system processing an entity's components
void processComponents(IAnnotatedComponent rootComponent) {
    if (rootComponent instanceof IAnnotatedComponentCollection) {
        IAnnotatedComponentCollection collection = (IAnnotatedComponentCollection) rootComponent;
        for (int i = 0; i < collection.componentCount(); i++) {
            IAnnotatedComponent child = collection.getComponent(i);
            if (child != null) {
                // Recursively process or apply logic
                processComponents(child);
            }
        }
    }
    // ... apply logic to the component itself
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Index Stability:** Do not rely on a component being at a fixed index. The internal order is not guaranteed by this contract and may change. Always query for components by type or capability, not by index.
- **Ignoring Nulls:** The getComponent method is nullable. Failure to check for a null return value will lead to NullPointerExceptions.
- **Direct Modification:** Avoid attempting to cast to a concrete collection type (like an ArrayList) to directly add or remove components at runtime. This breaks encapsulation and can lead to an unstable NPC state.

## Data Pipeline
This interface acts as a branching point in the component data flow. Systems that query an entity for its components will encounter these collections and must traverse them to find the specific leaf-node components they intend to operate on.

> Flow:
> AI Behavior System -> Queries NPC for Component -> Receives **IAnnotatedComponentCollection** -> Iterates Collection -> Finds and operates on Target IAnnotatedComponent

