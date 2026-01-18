---
description: Architectural reference for ActionOpenShop
---

# ActionOpenShop

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc
**Type:** Transient Behavior Object

## Definition
```java
// Signature
public class ActionOpenShop extends ActionBase {
```

## Architecture & Concepts
The ActionOpenShop class is a concrete implementation of the `ActionBase` contract within the server-side NPC Behavior System. It serves as a specialized command object whose sole responsibility is to bridge a successful NPC interaction with the player's UI framework.

Architecturally, this class represents a terminal node in an NPC's behavior tree or state machine. When an NPC's logic determines that an interaction should result in a shop opening, it invokes this action. ActionOpenShop encapsulates the logic required to identify the interacting player and instruct their client to render a specific shop interface, identified by a unique `shopId`. It does not manage the shop's content or transactions; it is purely a trigger mechanism.

This component is critical for creating interactive merchants and vendors in the game world, decoupling the NPC's core AI and decision-making from the specifics of the player's UI state.

## Lifecycle & Ownership
- **Creation:** An ActionOpenShop instance is not created directly via its constructor in gameplay code. It is instantiated by the server's NPC asset loading pipeline, specifically through a corresponding `BuilderActionOpenShop`. This builder reads configuration data from an NPC's definition file (e.g., a JSON asset) and constructs the action, injecting the configured `shopId`. The resulting object is then owned by the NPC's `Role` component.
- **Scope:** The lifetime of an ActionOpenShop instance is tightly bound to the NPC entity to which it is attached. It persists in memory as long as the NPC exists in the world.
- **Destruction:** The object is marked for garbage collection when its parent NPC entity is unloaded or destroyed. It does not manage any unmanaged resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The internal state is minimal and **immutable**. The class holds a single `final` field, `shopId`, which is populated at creation time and cannot be changed. The action itself is stateless with respect to its execution; all contextual data required for its operation (such as the target player) is passed as arguments to its methods by the NPC behavior system during the server tick.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. It interacts directly with entity components like `Player` and `PageManager`, which are not designed for concurrent access. Invoking its methods from an asynchronous task or a different thread will result in severe data corruption, race conditions, and server instability.

## API Surface
The public contract is defined by its parent, `ActionBase`, and is intended for invocation by the NPC system, not external code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true if the action can be performed. Critically, this requires that the NPC has a valid interaction target (a player). |
| execute(...) | boolean | O(1) | Triggers the shop UI to open for the interacting player. Asserts that the target player and their required components exist. Returns false if no interaction target is present. |

## Integration Patterns

### Standard Usage
This class is not intended for direct manual use. It is configured within an NPC's asset files and invoked automatically by the NPC's `Role` component as part of its behavior processing loop. The following conceptual example illustrates how the system uses an instance of this class.

```java
// Conceptual example of internal system usage
// An NPC's Role component iterates through its available actions.

Ref<EntityStore> npcRef = ...;
Role npcRole = ...;
InfoProvider sensorInfo = ...;
Store<EntityStore> worldStore = ...;
ActionBase currentAction = npcRole.getActiveAction(); // This might be an ActionOpenShop

if (currentAction.canExecute(npcRef, npcRole, sensorInfo, dt, worldStore)) {
    currentAction.execute(npcRef, npcRole, sensorInfo, dt, worldStore);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionOpenShop()`. The `shopId` and other internal state must be configured and injected by the asset loading system via its corresponding builder. Direct creation bypasses this essential configuration step.
- **Manual Invocation:** Do not retrieve this action from an NPC and call `execute` directly. This bypasses the `canExecute` precondition check, which ensures an interaction target is present. Calling `execute` without a valid target will result in assertion errors or a `NullPointerException`.
- **Asynchronous Execution:** Never pass an instance of this action to another thread for execution. All interactions with the entity component system must occur on the main server thread.

## Data Pipeline
ActionOpenShop acts as a key translation point in the player interaction data flow, converting a server-side game event into a client-side UI state change.

> Flow:
> Player Interaction Input -> Server Network Layer -> NPC Interaction System -> NPC Behavior Tree determines action -> **ActionOpenShop.execute()** -> Player.PageManager.openCustomPage() -> Server sends "Open UI" network packet -> Client Network Layer -> Client UI System renders ShopPage

