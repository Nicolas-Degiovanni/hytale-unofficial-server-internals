---
description: Architectural reference for ActionMount
---

# ActionMount

**Package:** com.hypixel.hytale.builtin.mounts.npc
**Type:** Transient

## Definition
```java
// Signature
public class ActionMount extends ActionBase {
```

## Architecture & Concepts
The ActionMount class is a server-side NPC behavior primitive that orchestrates the logic for a player mounting an NPC. It functions as a specific *Action* within the NPC's AI state machine, typically defined within an NPC's Role asset.

Its primary architectural role is to act as a state transition coordinator, modifying both the NPC and the Player entities to establish the "mounted" state. This is not a simple parent-child transformation; ActionMount performs a deep integration between the two entities:

1.  **NPC State Transition:** It attaches a persistent **NPCMountComponent** to the NPC entity. This component serves as the canonical record of the mounted state, holding references to the owning player and the anchor offset.
2.  **AI Suppression:** A critical design pattern employed here is the immediate request to change the NPC's role to a special, inert role, identified by the string constant **Empty_Role**. This effectively disables the NPC's standard AI, preventing it from walking, attacking, or performing other behaviors while it is being controlled by the player. The NPC becomes a passive vehicle.
3.  **Player Control Hijacking:** The action directly reaches into the Player's entity components and modifies their **MovementManager**. By applying a new **MovementConfig** asset, it fundamentally changes how player input is translated into motion, effectively giving the player control over the mount's movement profile (e.g., flight, higher speed, different jump physics).

This class is the nexus where the NPC AI System, the Entity Component System (ECS), and the Player Movement System converge to create the mount feature.

### Lifecycle & Ownership
-   **Creation:** ActionMount is instantiated by the server's asset loading pipeline, specifically by a **BuilderActionMount** when parsing an NPC's behavior definition from asset files. It is *not* intended to be created dynamically during gameplay.
-   **Scope:** An instance of ActionMount is stateless and reusable. It persists for the lifetime of the NPC's Role definition in memory. It represents the *capability* to be mounted, not the state of being mounted itself.
-   **Destruction:** The object is eligible for garbage collection when its parent Role definition is unloaded, typically during a server-wide asset reload or shutdown. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** This class is **immutable**. All of its internal fields (anchor coordinates, configuration IDs) are final and are initialized once during construction. It holds no mutable state related to a specific in-world NPC or player. All stateful operations are performed on components within the shared entity **Store**.
-   **Thread Safety:** The class itself is inherently thread-safe due to its immutability. However, its methods are designed to be called exclusively from the main server game loop thread, which guarantees serialized access to the **Store**. Calling its methods from an asynchronous context will break the server's ECS consistency and is strictly forbidden.

## API Surface
The public contract is defined by its parent, ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Verifies that the action can be performed. Checks for a valid, living interaction target (the player). This must be called before execute. |
| execute(...) | boolean | O(1) | Executes the mounting logic. This method is not idempotent and will fail if the NPC is already mounted. It performs significant state changes across multiple entities. |

## Integration Patterns

### Standard Usage
This class is not invoked directly in code. It is configured within an NPC's asset definition and triggered by the NPC's AI system when conditions are met (e.g., a player interacts with the NPC). The system handles the invocation.

A conceptual view of the trigger flow:

```java
// PSEUDO-CODE: Represents the NPC AI loop
// This code does not exist; it illustrates the concept.

Role currentRole = npc.getActiveRole();
ActionBase action = currentRole.findActionFor(InteractionEvent.class);

if (action instanceof ActionMount) {
    if (action.canExecute(npcRef, currentRole, ...)) {
        action.execute(npcRef, currentRole, ...);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ActionMount()`. The object must be constructed by the server's asset builders to ensure its configuration is valid.
-   **Stateful Reuse:** While the object is reusable, do not attempt to store instance-specific data on it. It is a stateless service object.
-   **Invalid Configuration:** Defining an NPC asset that uses ActionMount but provides an invalid `movementConfigId` will result in a runtime failure when the action is executed. The server will fail to find the asset and the player's movement will not be updated correctly.
-   **Skipping Pre-conditions:** Calling `execute` without a preceding successful call to `canExecute` can lead to `NullPointerException`s or assertion failures, as `execute` assumes a valid, living player target exists.

## Data Pipeline
ActionMount does not process a stream of data. Instead, it represents a command that triggers a series of state mutations within the Entity Component System.

> **Control Flow:**
>
> Player Interaction Event → NPC AI determines interaction is a mount request → **ActionMount.canExecute()** returns true → **ActionMount.execute()** is called → The following mutations occur:
>
> 1.  **NPC Entity:** An `NPCMountComponent` is added via the `Store`.
> 2.  **NPCMountComponent:** The component is populated with the Player's reference and anchor coordinates.
> 3.  **RoleChangeSystem:** A request is dispatched to change the NPC's role to `Empty_Role`, disabling its AI.
> 4.  **Player Entity:** The Player's `MovementManager` component is retrieved.
> 5.  **MovementManager:** The manager's default settings are overwritten with a new `MovementConfig` loaded from assets, and these new settings are immediately applied and synchronized with the client.

