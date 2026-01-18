---
description: Architectural reference for EntityGroup
---

# EntityGroup

**Package:** com.hypixel.hytale.server.core.entity.group
**Type:** Transient Component

## Definition
```java
// Signature
public class EntityGroup implements Component<EntityStore> {
```

## Architecture & Concepts
The EntityGroup is a server-side data component within the Entity-Component-System (ECS) framework. It is not a standalone manager or service; rather, it is a piece of state attached to an entity to represent a logical collection of other entities. Common use cases include player parties, NPC squads, or herds of animals.

The core responsibility of this class is to maintain a list of entity members and provide efficient mechanisms for iteration and membership checking. It employs a dual-collection strategy for performance:
1.  A **HashSet** (memberSet) provides O(1) average time complexity for adding, removing, and checking for the existence of a member.
2.  An **ObjectArrayList** (memberList) provides a stable, ordered list for fast iteration, which is critical for broadcasting events or updates to all members.

All entities are tracked via a Ref<EntityStore>, which is a safe, indirect reference. This prevents memory leaks and allows the system to gracefully handle cases where a member entity is destroyed or unloaded. Iteration logic within EntityGroup correctly checks if a reference is still valid before operating on it.

## Lifecycle & Ownership
-   **Creation:** An EntityGroup is not instantiated directly via its constructor. It is created and attached to an entity through the ECS framework, typically via a call like `entity.addComponent(EntityGroup.class)`. This ensures it is correctly registered within the game world.
-   **Scope:** The lifecycle of an EntityGroup instance is strictly tied to the entity it is attached to. It persists as long as its owning entity exists in the world.
-   **Destruction:** The component is marked for garbage collection when its owning entity is destroyed. The `clear` method can be invoked to manually dissolve the group, severing all references to its members and marking it as inactive. This is critical for preventing stale state if the group is disbanded but the owning entity persists.

## Internal State & Concurrency
-   **State:** The EntityGroup is a highly mutable state container. Its internal collections are frequently modified as entities join or leave the group. It holds direct references (via the Ref wrapper) to other game-state objects (entities).
-   **Thread Safety:** **This class is not thread-safe.** All methods that modify or read its state, such as `add`, `remove`, and the `forEachMember` family of iterators, must be called from the main server game loop thread. Unsynchronized access from other threads will result in `ConcurrentModificationException` or other undefined behavior.

## API Surface
The public API is designed for managing group membership and executing logic across the members.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(Ref<EntityStore> reference) | void | O(1) | Adds an entity to the group. Throws IllegalStateException if the entity is already a member. |
| remove(Ref<EntityStore> reference) | void | O(N) | Removes an entity from the group. Throws IllegalStateException if the entity is not a member. |
| isMember(Ref<EntityStore> reference) | boolean | O(1) | Checks if an entity is part of the group. |
| forEachMember(...) | void | O(N) | A family of methods to iterate over all members and apply a given function. |
| forEachMemberExcludingSelf(...) | void | O(N) | Iterates over all members except for the specified "sender" or "self" entity. |
| forEachMemberExcludingLeader(...) | void | O(N) | Iterates over all members except for the designated group leader. |
| setDissolved(boolean dissolved) | void | O(1) | Sets a flag to logically disable the group without clearing its member list immediately. |
| clear() | void | O(N) | Removes all members, nullifies the leader, and marks the group as dissolved. |

## Integration Patterns

### Standard Usage
An EntityGroup should always be retrieved from an existing entity. Logic is then performed on the retrieved component.

```java
// Assume 'partyLeaderEntity' is an entity with an EntityGroup component
EntityGroup party = partyLeaderEntity.getComponent(EntityGroup.class);

// Always check for null and dissolved state before use
if (party != null && !party.isDissolved()) {
    // Add a new player's entity reference to the party
    party.add(newPlayerEntity.getRef());

    // Broadcast a "welcome" message to all members except the new player
    party.forEachMemberExcludingSelf((member, sender, message) -> {
        // Logic to send the message to the 'member' entity
        MessageSystem.sendMessage(member, message);
    }, newPlayerEntity.getRef(), "Welcome to the party!");
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EntityGroup()`. Components created this way are not managed by the engine and will not function correctly. They must be added to an entity via the appropriate ECS manager.
-   **Asynchronous Modification:** Do not modify the group from a network thread, a separate task, or any context other than the main server tick. This will corrupt the internal state of the collections.
-   **Ignoring Dissolved State:** Failing to check `isDissolved()` before iterating can lead to performing logic on a group that is no longer considered active by the game systems.
-   **Storing Member List:** Do not get the member list via `getMemberList()` and store it long-term. The list is live and can be modified by other parts of the code, leading to stale data or concurrency issues if your stored copy is iterated later.

## Data Pipeline
EntityGroup acts as a stateful container and a dispatcher, not a data transformer. It is typically acted upon by game logic systems and, in turn, initiates actions on its member entities.

> **Flow 1: Group Formation**
> Player Input (Invite Command) -> Command Handling System -> Game Logic -> **EntityGroup.add()**

> **Flow 2: Event Broadcasting**
> Game Event (e.g., Leader uses a skill) -> Skill System -> **EntityGroup.forEachMemberExcludingLeader()** -> Effect System -> Apply Buff to Member Entity State

