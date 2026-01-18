---
description: Architectural reference for ActionFlockBeacon
---

# ActionFlockBeacon

**Package:** com.hypixel.hytale.server.flock.corecomponents
**Type:** Transient

## Definition
```java
// Signature
public class ActionFlockBeacon extends ActionBase {
```

## Architecture & Concepts
ActionFlockBeacon is a concrete implementation of the `ActionBase` class, representing a single, executable command within the server-side NPC AI framework. It operates strictly within the Entity Component System (ECS).

Its primary function is to facilitate intra-group communication for socially-aware NPCs. When executed, this action broadcasts a named message, or "beacon," to other entities belonging to the same flock. This allows for coordinated group behaviors, such as a scout NPC alerting its pack to danger or a leader signaling a new point of interest.

The action identifies the flock through the `FlockMembership` component on the executing entity. It then uses the flock's central `EntityGroup` component to resolve the list of members. Finally, it posts the message to the `BeaconSupport` component of each target member, where it can be detected by other AI sensors. The action supports various targeting modes, including broadcasting to all members, only the leader, or excluding the sender.

## Lifecycle & Ownership
-   **Creation:** ActionFlockBeacon is instantiated exclusively by its corresponding builder, `BuilderActionFlockBeacon`. This process is typically driven by the server's asset loading system, which parses NPC behavior definitions (e.g., from JSON or HOCON files) and constructs the necessary action objects.
-   **Scope:** An instance of this class is part of a static, shared behavior graph (like a Behavior Tree or State Machine) for a specific NPC type. It is configured once upon creation and its state is immutable thereafter. It persists for the lifetime of the server or until game assets are reloaded. It is not created per-NPC instance.
-   **Destruction:** The object is garbage collected when its parent behavior graph is discarded, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** The internal state of ActionFlockBeacon is **Immutable**. All configuration fields (`message`, `expirationTime`, `sendToSelf`, etc.) are marked as final and are initialized only once in the constructor. The object itself does not cache or modify any data during execution; it only reads from and writes to the ECS `Store`.

-   **Thread Safety:** This class is **Conditionally Thread-Safe**. Its immutability makes it inherently safe to be shared and executed by multiple threads. However, its safety is contingent upon the guarantees provided by the ECS `Store` passed into its methods. The server's NPC update loop must ensure that component data for a given entity is not being modified concurrently by another thread during the execution of this action. The action itself introduces no new synchronization primitives or thread-safety risks.

## API Surface
The public contract is defined by its parent `ActionBase` and is intended for use by the internal AI scheduler, not by general-purpose game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Predicate to determine if the action can run. Checks for the existence of the `FlockMembership` component and validates target slot data. |
| execute(...) | boolean | O(N) | Executes the broadcast logic. Complexity is linear, where N is the number of members in the target flock. |

## Integration Patterns

### Standard Usage
This action is not designed to be invoked directly. It is defined within an NPC's behavior asset file and is executed automatically by the NPC's `Role` or an equivalent AI driver during the server tick.

The following conceptual example shows how the *system* would invoke the action on an entity.

```java
// This logic is handled by the server's NPC update loop.
// Do not replicate this pattern.

// Assume 'npcRef' is the entity executing the action and 'npcRole' is its AI driver.
// Assume 'action' is an instance of ActionFlockBeacon from the NPC's behavior tree.
Store<EntityStore> worldStore = server.getWorldStore();
InfoProvider sensorInfo = npcRole.getSensorInfo();
double deltaTime = server.getDeltaTime();

if (action.canExecute(npcRef, npcRole, sensorInfo, deltaTime, worldStore)) {
    action.execute(npcRef, npcRole, sensorInfo, deltaTime, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new ActionFlockBeacon()`. The object is complex to configure and must be created via `BuilderActionFlockBeacon` during asset loading to ensure its internal state is valid.
-   **Execution Outside AI Loop:** Calling `execute` outside the server's managed NPC update tick is unsafe. The `Store` and `Ref` arguments may be invalid or point to stale data, leading to unpredictable behavior or exceptions.
-   **State Modification:** Do not attempt to modify the internal fields of this class after construction (e.g., via reflection). Its immutability is critical for safe concurrent execution across multiple NPC instances sharing the same behavior definition.

## Data Pipeline
This action functions as a data producer within the ECS, creating messages that are consumed by other AI components.

> Flow:
> NPC Behavior System triggers action -> **ActionFlockBeacon.execute()** -> Reads `FlockMembership` from self -> Reads `EntityGroup` from flock entity -> Iterates members -> Writes message to target entity's `BeaconSupport` component -> AI Sensor on target entity reads message from its own `BeaconSupport` component.

