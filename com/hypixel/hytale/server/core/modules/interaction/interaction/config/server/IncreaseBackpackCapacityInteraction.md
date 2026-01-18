---
description: Architectural reference for IncreaseBackpackCapacityInteraction
---

# IncreaseBackpackCapacityInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class IncreaseBackpackCapacityInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The IncreaseBackpackCapacityInteraction class is a server-authoritative, data-driven implementation of a game interaction. It represents a single, atomic action that modifies a player's state—specifically, increasing the size of their backpack inventory.

Architecturally, this class is not a service or a manager but a **configurable command object**. Its behavior and properties are defined externally in game data files and deserialized at runtime by the engine's codec system. The static `CODEC` field is the cornerstone of this design, defining the schema for how this interaction is loaded from configuration. It specifies the `Capacity` property, its data type (short), and validation rules (must be at least 1).

By extending SimpleInstantInteraction, it signals to the interaction system that this action is immediate and non-continuous. It executes its logic once in the `firstRun` method and does not persist any state related to the interaction itself. Its primary role is to encapsulate the logic for a specific game rule, decoupling the core interaction system from the myriad of possible in-game actions.

## Lifecycle & Ownership
- **Creation:** Instances are not created directly using the `new` keyword. They are instantiated by the Hytale engine's asset loading pipeline, which uses the static `CODEC` field to deserialize the object from a game configuration file (e.g., an item definition JSON).
- **Scope:** The object's lifetime is tied to the asset it is a part of. For example, if this interaction is attached to a specific item, an instance of this class will exist in memory as long as that item's definition is loaded by the server. It is effectively a stateless data container once loaded.
- **Destruction:** The object is marked for garbage collection when the server unloads the associated game assets, such as during a server shutdown or when a zone or mod is unloaded.

## Internal State & Concurrency
- **State:** The internal state, specifically the `capacity` field, is **effectively immutable** after deserialization. It is set once during the asset loading process and is not designed to be modified at runtime. The class itself holds no mutable state related to a specific player or interaction event.
- **Thread Safety:** The object instance is inherently thread-safe due to its immutable state. However, the execution of its methods, particularly `firstRun`, is **not thread-safe** by design. This method modifies shared game state (Player components, Inventory). The engine's interaction module is responsible for ensuring that `firstRun` is invoked exclusively on the main server thread (the game tick) to prevent race conditions and ensure transactional integrity of game state modifications.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Returns `Server`, indicating this is a server-authoritative action that does not require further client-side data. |
| firstRun(...) | void | O(1) | Executes the core logic. Retrieves the Player component from the InteractionContext, increases their backpack capacity, sends a confirmation message, and consumes the item that triggered the interaction. |

## Integration Patterns

### Standard Usage
A developer or content creator does not interact with this class directly in code. Instead, they define it declaratively within a game asset file. The server's interaction system then invokes it automatically in response to player actions.

The following example is a conceptual representation of how the engine might trigger this interaction.

```java
// Conceptual Engine Code
// This code is NOT meant to be written by a user of the class.

// Player right-clicks an item
void onPlayerUseItem(Player player, ItemStack item) {
    // Engine retrieves the interaction definition from the item's data
    IncreaseBackpackCapacityInteraction interaction = item.getInteraction(IncreaseBackpackCapacityInteraction.class);

    if (interaction != null) {
        // Engine builds the context and invokes the interaction
        InteractionContext context = buildContextForPlayer(player);
        interaction.firstRun(InteractionType.PRIMARY, context, player.getCooldownHandler());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IncreaseBackpackCapacityInteraction()`. The `capacity` field would be uninitialized, and the object would not be correctly registered within the game's content system. All instances must be created via the engine's asset loader and the `CODEC`.
- **External State Mutation:** Do not attempt to modify the `capacity` field after the object has been loaded. This violates the data-driven design and can lead to unpredictable behavior.
- **Off-Thread Execution:** Never call the `firstRun` method from any thread other than the main server game loop. Doing so will cause severe concurrency issues, data corruption, and server instability.

## Data Pipeline
The flow of data and control for this interaction follows a clear server-authoritative pattern.

> Flow:
> Player Input (Client) -> Network Packet (Use Item) -> Server Network Handler -> **Interaction Module** -> Retrieves `IncreaseBackpackCapacityInteraction` from Item Definition -> **`firstRun` method execution** -> Modifies `Player` and `Inventory` components -> State change is synchronized back to the client via the entity component system.

