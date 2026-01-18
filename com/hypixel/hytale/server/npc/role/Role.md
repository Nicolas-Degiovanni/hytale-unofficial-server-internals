---
description: Architectural reference for Role
---

# Role

**Package:** com.hypixel.hytale.server.npc.role
**Type:** Stateful Component

## Definition
```java
// Signature
public class Role implements IAnnotatedComponentCollection {
```

## Architecture & Concepts

The Role class is the central nervous system for a server-side Non-Player Character (NPC). It encapsulates the entirety of an NPC's behavior, decision-making logic, and physical interaction with the world. It is not merely a data container; it is an active, stateful component that orchestrates all aspects of the NPC's AI.

Architecturally, Role acts as a high-level controller that integrates numerous specialized sub-systems. It follows a strong **Composition over Inheritance** pattern, delegating specific domains of logic to dedicated "Support" objects:
-   **StateSupport:** Manages high-level states (e.g., idle, combat, fleeing) and transitions between them.
-   **CombatSupport:** Governs all combat-related behaviors, including target acquisition, attack patterns, and damage calculations.
-   **WorldSupport:** Handles environmental queries, such as line-of-sight checks and block sensing.
-   **EntitySupport:** Manages the NPC's own entity state, including motion and animation commands.
-   **MarkedEntitySupport:** Tracks references to other significant entities, such as targets or allies.

The core of the Role's decision-making is a behavior tree, represented by the **rootInstruction**. This tree is traversed each tick to determine the NPC's actions. The Role translates the abstract outputs of this tree (e.g., "move towards target") into concrete movement commands by manipulating **Steering** vectors. These vectors are then processed by a **MotionController**, which simulates the NPC's physical movement model (e.g., walking, flying).

A Role is defined by data-driven assets, configured via a **BuilderRole**. This design decouples the complex AI logic within the Role class from the specific parameters that define a unique NPC type, allowing for wide behavioral variety without code changes.

## Lifecycle & Ownership

-   **Creation:** A Role is instantiated deep within the NPC creation pipeline. A corresponding **BuilderRole** asset is loaded, and its data is passed to the Role constructor along with a **BuilderSupport** context object. This process is managed by the server's entity spawning systems; direct instantiation is an anti-pattern.
-   **Scope:** The lifecycle of a Role is strictly bound to its owning **NPCEntity**. It persists as long as the NPC exists in the world. If an NPC's role is changed dynamically, the existing Role instance is destroyed and a new one is created.
-   **Destruction:** The Role is marked for garbage collection when its parent NPCEntity is removed from the world. The **unloaded** and **removed** methods serve as explicit destruction hooks, responsible for clearing internal state, detaching from other systems, and ensuring no resource leaks occur.

## Internal State & Concurrency

-   **State:** The Role class is highly mutable and stateful. It maintains the current state of the AI, including the active node in the behavior tree, target references, steering vectors, cached environmental data via **PositionCache**, and a set of boolean flags for custom state logic. This internal state is critical for behavior continuity between ticks.
-   **Thread Safety:** This class is **not thread-safe**. It is designed to be accessed and manipulated exclusively by the main server game loop thread. All state modifications and method calls, particularly the **tick** method, must occur within the synchronized server tick. Unmanaged multi-threaded access will lead to severe race conditions, data corruption, and unpredictable AI behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(ref, tickTime, store) | void | O(N) | The primary entry point for the AI update cycle. Executes the behavior tree and computes steering. N is the complexity of the instruction tree and the number of nearby entities. |
| spawned(holder, npcComponent) | void | O(1) | Lifecycle hook called once the NPC is fully present in the world. Initializes inventories and state. |
| unloaded() | void | O(1) | Lifecycle hook called when the NPC is being removed. Cleans up internal state and resources. |
| setActiveMotionController(name) | boolean | O(1) | Switches the underlying physical movement model (e.g., from walking to flying). Critical for NPCs with multiple locomotion types. |
| addForce(velocity, config) | void | O(1) | Applies an external force to the NPC. This is the primary integration point for physics interactions like knockback. |
| setFlag(index, value) | void | O(1) | Sets a boolean flag used for custom logic within the behavior tree. Throws if flags are not allocated or the index is out of bounds. |
| blendAvoidance(ref, position, steering, buffer) | void | O(K) | Modifies a steering vector to avoid collisions with nearby entities. K is the number of entities in the avoidance range. |
| blendSeparation(ref, position, steering, buffer) | void | O(K) | Modifies a steering vector to maintain separation from flockmates or other friendly entities. K is the number of entities in the avoidance range. |

## Integration Patterns

### Standard Usage

A Role is never managed directly. It is an internal component of an NPCEntity. Other game systems interact with the Role *through* the NPCEntity, typically to influence its state or movement. The engine itself is the primary caller, invoking the **tick** method once per server update.

```java
// External systems (e.g., a combat handler) typically influence the Role
// by applying forces, rather than setting velocity directly.
NPCEntity npc = ...;
Role npcRole = npc.getRole();

// Apply knockback force from an explosion
Vector3d knockback = new Vector3d(x, y, z);
npcRole.addForce(knockback, knockbackConfig);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new Role()`. Roles must be constructed by the server's asset and entity loading systems to ensure they are correctly configured and initialized.
-   **External State Mutation:** Do not directly modify public fields like **bodySteering**. Use API methods like **addForce** to influence movement. Direct manipulation will be overwritten by the AI on the next tick and can break complex behaviors like avoidance.
-   **Multi-threaded Access:** Never read from or write to a Role instance from any thread other than the main server thread. This will cause unpredictable behavior and server instability.
-   **Re-using Instances:** A Role instance is tied to a single NPCEntity for its entire lifetime. Do not attempt to re-assign a Role from one NPC to another.

## Data Pipeline

The Role processes a flow of data each tick to produce a desired movement output.

> Flow:
> World State (Entity Positions, Terrain) -> **PositionCache** -> Behavior Tree (**rootInstruction**) -> Motion Command (**setNextBodyMotionStep**) -> Raw Steering Vector -> **blendSeparation / blendAvoidance** -> Final Steering Vector -> **MotionController** -> Physics Velocity Update

