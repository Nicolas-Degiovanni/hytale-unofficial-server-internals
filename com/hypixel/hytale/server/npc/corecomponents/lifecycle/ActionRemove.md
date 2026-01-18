---
description: Architectural reference for ActionRemove
---

# ActionRemove

**Package:** com.hypixel.hytale.server.npc.corecomponents.lifecycle
**Type:** Transient Component

## Definition
```java
// Signature
public class ActionRemove extends ActionBase {
```

## Architecture & Concepts
ActionRemove is a concrete implementation of the ActionBase class, designed to perform a single, destructive operation: removing an entity from the game world. It is a fundamental component within the server-side NPC Behavior System, acting as a terminal node in a behavior tree or a final state in a state machine.

The component's primary architectural feature is its configurable targeting system, governed by the internal **useTarget** flag.
- When **useTarget** is false, the action targets the NPC entity that owns this component, effectively making it a self-destruct mechanism.
- When **useTarget** is true, the action targets an entity identified by the NPC's sensor system, provided through the InfoProvider.

A critical, non-negotiable design constraint is baked into this component: it will **never** remove an entity that is a Player. This hard-coded safety mechanism prevents catastrophic game state corruption where an NPC's logic could inadvertently remove a connected player from the server.

## Lifecycle & Ownership
- **Creation:** ActionRemove is instantiated exclusively by the server's NPC asset loading pipeline. Its configuration, including the **useTarget** flag, is defined declaratively in an NPC's asset files. It is constructed via its corresponding builder, BuilderActionRemove, during server startup or when NPC assets are hot-loaded.
- **Scope:** The lifecycle of an ActionRemove instance is tightly coupled to the NPC entity that owns it. It exists as part of the NPC's behavior definition, typically held within a Role component, and persists for the entire lifetime of that NPC.
- **Destruction:** The object is eligible for garbage collection when its parent NPC entity is removed from the world and all references to its components are released. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The component's state is **Immutable** after construction. The core behavioral flag, **useTarget**, is a final field initialized in the constructor. The class itself does not cache or manage any dynamic game state; it operates purely on the state passed into its methods during each game tick.
- **Thread Safety:** This class is **Not Thread-Safe** and is designed to be operated exclusively by the main server game loop thread. Its methods manipulate the core EntityStore, which is not a thread-safe collection. All invocations must be synchronized with the server's tick cycle to prevent race conditions and data corruption.

## API Surface
The public API is minimal and conforms to the contract established by ActionBase, intended for invocation by the NPC behavior system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Evaluates if the removal can proceed. Checks superclass conditions and, if targeting, verifies a valid target exists. |
| execute(...) | boolean | O(1) | Performs the entity removal. Identifies the target, verifies it is not a Player, and issues a removal command to the EntityStore. |

## Integration Patterns

### Standard Usage
This component is not intended for direct developer interaction. It is configured within an NPC's asset definition and executed by the behavior engine. The following example illustrates the conceptual invocation by a hypothetical behavior tree node.

```java
// Executed by an internal NPC Behavior Engine
// An instance of ActionRemove is held by the active behavior node.

boolean canProceed = this.actionRemove.canExecute(npcRef, npcRole, sensorInfo, deltaTime, worldStore);

if (canProceed) {
    this.actionRemove.execute(npcRef, npcRole, sensorInfo, deltaTime, worldStore);
    // The entity is now queued for removal.
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new ActionRemove()`. This bypasses the asset pipeline and the builder system, resulting in a misconfigured and non-functional component. All actions must be defined in asset files.
- **Ignoring Pre-conditions:** Calling `execute` without a preceding successful call to `canExecute` is a violation of the component's contract. This can lead to NullPointerExceptions if `useTarget` is true and no valid sensor information is available.
- **Player Removal Attempts:** Do not design NPC logic that intentionally targets a Player with this action. While the internal guard will prevent the removal, such a design indicates a misunderstanding of the component's purpose and creates useless processing overhead.

## Data Pipeline
ActionRemove acts as a data sink, terminating an entity's lifecycle based on inputs from the NPC's state and sensors. It does not produce or transform data.

> Flow:
> NPC Behavior Engine -> **ActionRemove.canExecute()** -> **ActionRemove.execute()** -> EntityStore.removeEntity() -> World State Update

