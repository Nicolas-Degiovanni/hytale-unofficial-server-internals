---
description: Architectural reference for ActionRole
---

# ActionRole

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient Command Object

## Definition
```java
// Signature
public class ActionRole extends ActionBase {
```

## Architecture & Concepts
The ActionRole class is a concrete implementation of the Command Pattern, operating within the server-side NPC Behavior System. It represents a single, immutable action that instructs an NPC entity to transition from its current Role to a new one.

Its primary architectural function is to decouple the *intent* to change a role from the *mechanism* that performs the change. This class encapsulates all necessary parameters for the transition—such as the target role, appearance settings, and initial state—which are defined declaratively in NPC asset files.

ActionRole is designed as a **terminal action**. When its `execute` method is called, it signals to the NPC's current Role that its behavior sequence is complete. The actual role transition is not performed by this class directly; instead, it submits a request to the centralized **RoleChangeSystem**, which processes the change in a subsequent server tick. This ensures that role transitions are managed systemically and do not occur mid-tick, preventing state corruption.

## Lifecycle & Ownership
- **Creation:** ActionRole instances are not created dynamically during gameplay. They are instantiated once by the asset pipeline when an NPC's behavior definition is loaded into memory. The `BuilderActionRole` class is responsible for parsing the asset configuration and constructing the corresponding ActionRole object.
- **Scope:** An instance of ActionRole persists for the entire lifetime of its parent NPC asset definition. As it is immutable, a single instance can be shared and reused by all NPC entities of the same type.
- **Destruction:** The object is eligible for garbage collection only when the server unloads the corresponding NPC asset definition, for example, during a zone change or server shutdown.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields, such as `roleIndex` and `kind`, are final and are initialized exclusively in the constructor. The object's state cannot be modified after creation. This design is critical for its role as a reusable, data-driven command.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. Its methods can be safely invoked from any thread.
    - **WARNING:** While the object itself is thread-safe, the `execute` method interacts with mutable game state (EntityStore, Role). It is imperative that `execute` is only ever called from the main server thread that owns the entity world to prevent race conditions and data corruption.

## API Surface
The public contract is defined by its parent `ActionBase` and is invoked by the NPC behavior processor.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Checks if the action can be performed. Returns false if the target role index is invalid. |
| execute(...) | boolean | O(1) | Submits a role change request to the RoleChangeSystem. Returns false if a change is already pending. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly in Java code. Instead, it is configured within an NPC's asset files (e.g., JSON or HOCON). The NPC's behavior system (such as a state machine or behavior tree) invokes the action at the appropriate time.

The following example demonstrates how the system would execute a pre-loaded action.

```java
// Context: Inside an NPC behavior processing loop
// An 'action' instance would be retrieved from the NPC's loaded behavior graph.

ActionRole changeToGuardRole = npc.getBehaviorAction("becomeGuard");

// The system checks and executes the action for a specific entity
if (changeToGuardRole.canExecute(entityRef, currentRole, sensorInfo, dt, store)) {
    changeToGuardRole.execute(entityRef, currentRole, sensorInfo, dt, store);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ActionRole()`. The object must be constructed by the asset loading pipeline via its corresponding builder to ensure its internal state is correctly initialized from game data.
- **External Invocation:** Do not call `execute` from arbitrary game logic. This action is designed to be managed exclusively by an NPC's core behavior processor to maintain state integrity. Bypassing the processor can lead to corrupted AI state.
- **State Assumption:** Do not assume the role change is instantaneous. The `execute` method only *requests* a change. The actual transition is handled later in the server tick by the `RoleChangeSystem`. Any logic that depends on the new role must wait for the transition to complete.

## Data Pipeline
ActionRole acts as a trigger in a control-flow pipeline, not a data-transformation pipeline. It initiates a state change that propagates through several core server systems.

> Flow:
> NPC Behavior Processor (evaluates state) -> **ActionRole.execute()** -> RoleChangeSystem (queues request) -> Server Tick End -> RoleChangeSystem (processes request) -> Entity Component Update (new Role attached) -> Next Tick AI Update (new Role logic runs)

