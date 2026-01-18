---
description: Architectural reference for ActionOpenBarterShop
---

# ActionOpenBarterShop

**Package:** com.hypixel.hytale.builtin.adventure.npcshop.npc
**Type:** Transient

## Definition
```java
// Signature
public class ActionOpenBarterShop extends ActionBase {
```

## Architecture & Concepts
ActionOpenBarterShop is a concrete implementation of the ActionBase class, designed to function as a terminal operation within an NPC's behavior logic. In the context of the server's AI and NPC systems, this class represents a single, atomic action: instructing a player's client to open a specific bartering interface.

This class acts as a critical bridge between the server-side NPC simulation and the client-side user interface. When an NPC's behavior tree or state machine determines that a shop interface should be shown to an interacting player, it invokes this action. The action's primary responsibility is to identify the target player, construct a new BarterPage instance configured with a specific shop identifier, and push this page to the player's PageManager. This triggers the necessary server-to-client communication to render the shop UI for the player.

The core piece of configuration for this class is the **shopId**, a string that uniquely identifies a set of trade definitions loaded from game assets. This decouples the NPC's behavior logic from the specific contents of the shop, allowing designers to change trade offers without modifying the NPC's core AI.

### Lifecycle & Ownership
- **Creation:** Instances are created by the server's NPC asset loading pipeline. A corresponding BuilderActionOpenBarterShop reads configuration from an NPC definition file (e.g., a JSON asset) and constructs an ActionOpenBarterShop instance. This occurs when the server loads and parses its NPC definitions at startup or during a hot-reload.
- **Scope:** The object's lifetime is tied to the loaded NPC Role definition. It is a stateless, reusable command object that exists as part of an NPC's behavioral template. It is not instantiated per-NPC-entity, but rather shared by all NPCs that use the same role.
- **Destruction:** The object is eligible for garbage collection when its parent NPC Role definition is unloaded by the server.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its only state, the shopId, is a final field initialized during construction. It does not cache data or modify its internal state during execution.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be executed exclusively on the main server thread as part of the game's tick cycle. Its methods interact directly with the EntityStore and other core game components which are not designed for concurrent access. All invocations must be synchronized with the main game loop.

## API Surface
The public contract is defined by its parent, ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true if the NPC has a valid interaction target (a player). This is a precondition check before execution. |
| execute(...) | boolean | O(1) | Retrieves the target player and instructs their PageManager to open the barter shop UI. Returns true on success. |

## Integration Patterns

### Standard Usage
This class is not intended for direct, procedural invocation by developers. It is designed to be configured within an NPC asset file and executed by the NPC Role and behavior systems. The engine handles the lifecycle and invocation.

A conceptual asset definition might look like this:

```yaml
# Example NPC Asset (conceptual)
id: "village_merchant"
role: "shopkeeper"
actions:
  - type: "ActionOpenBarterShop"
    shopId: "general_store_wares"
```

The engine parses this, uses BuilderActionOpenBarterShop to create the instance, and integrates it into the shopkeeper role's behavior tree.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not call `new ActionOpenBarterShop()`. The class requires its corresponding builder for correct initialization from asset data. Manual creation bypasses the asset pipeline and will result in a non-functional object.
- **External Invocation:** Do not call the execute method from outside the NPC's state processing loop. The method relies on contextual state, such as the `interactionIterationTarget` within the Role, which is only guaranteed to be valid during an NPC's update tick.

## Data Pipeline
This action serves as a trigger point in a server-to-client data flow.

> Flow:
> Player Interaction Event -> Server NPC Behavior Processor -> **ActionOpenBarterShop.execute()** -> Player.PageManager.openCustomPage() -> Network Packet (Open UI) -> Client UI System -> Render BarterPage

