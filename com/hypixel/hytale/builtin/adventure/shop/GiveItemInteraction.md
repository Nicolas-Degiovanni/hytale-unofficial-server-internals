---
description: Architectural reference for GiveItemInteraction
---

# GiveItemInteraction

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Transient Command Object

## Definition
```java
// Signature
public class GiveItemInteraction extends ChoiceInteraction {
```

## Architecture & Concepts
The GiveItemInteraction class is a concrete implementation of the Command design pattern, encapsulated within the server's interaction framework. It represents a single, atomic action: granting a specified quantity of an item to a player. This class is not a long-lived service; rather, it is a data-driven object that is instantiated to execute a specific, pre-configured task.

Its primary role is to bridge declarative game content (defined in asset files) with the imperative logic of the server's Entity Component System (ECS). The static CODEC field is the cornerstone of this design. It allows the game engine to deserialize a configuration block—for example, from a JSON or HOCON file defining a shop's inventory or a quest reward—directly into an executable GiveItemInteraction instance.

This approach fully decouples the action's logic from its configuration. Game designers can create or modify interactions that grant items without writing any Java code, simply by editing the corresponding asset files. The `run` method is the command's execution phase, directly mutating the game state by interacting with the player's inventory component.

## Lifecycle & Ownership
- **Creation:** A GiveItemInteraction instance is almost exclusively created by the server's asset loading and codec system. During server startup or asset hot-reloading, the static CODEC is used to parse a data structure and instantiate the object. Programmatic creation via `new GiveItemInteraction()` is possible but is a secondary, less common use case reserved for dynamically generated interactions.
- **Scope:** The object's lifecycle is extremely brief and tied to a specific player action. It is typically held as a member of a larger configuration object, such as a ChoicePage, and exists only to be executed once. After its `run` method is invoked, it is no longer needed and becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** The internal state, consisting of `itemId` and `quantity`, is mutable only during deserialization by the CODEC. After construction, the object is treated as effectively immutable. Its purpose is to hold static data for a single execution, not to manage a changing state.
- **Thread Safety:** **This class is not thread-safe and must not be treated as such.** The `run` method performs direct, unsynchronized mutations on a player's inventory, which is a critical part of the server's state. All interactions with this class, particularly the invocation of `run`, must be performed on the main server game thread to prevent catastrophic race conditions and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(store, ref, playerRef) | void | O(1) | Executes the interaction. Retrieves the player component and adds the configured ItemStack to their inventory. Must be called from the main game thread. |
| getItemId() | String | O(1) | Returns the unique identifier of the item to be granted. |
| getQuantity() | int | O(1) | Returns the number of items to be granted. |

## Integration Patterns

### Standard Usage
This object is not intended to be retrieved from a service locator. Instead, it is typically part of a larger data structure, such as a list of choices presented to a player. The system invokes the `run` method in response to a player's input.

```java
// Assume 'choicePage' is an object deserialized from game assets
// and 'selectedInteraction' is the GiveItemInteraction instance
// associated with the button the player clicked.

ChoiceInteraction selectedInteraction = choicePage.getInteractionForChoice(playerChoiceId);

if (selectedInteraction instanceof GiveItemInteraction) {
    // The engine provides the necessary ECS context (store, ref, playerRef)
    // when handling the player's action.
    selectedInteraction.run(server.getStore(), player.getRef(), player.getPlayerRef());
}
```

### Anti-Patterns (Do NOT do this)
- **State Mutation:** Do not modify the `itemId` or `quantity` fields after the object has been constructed. It is a data-carrier, not a stateful object. Modifying it post-creation leads to unpredictable behavior.
- **Off-Thread Execution:** Never call the `run` method from an asynchronous task or a different thread. All game state mutations must be synchronized with the server's main tick loop.
- **Re-use:** Do not hold a reference to a GiveItemInteraction instance for re-use. It is designed as a single-use command object.

## Data Pipeline
The primary flow for this class involves deserialization from a static asset file and subsequent execution, which mutates the live game state.

> Flow:
> Game Asset (e.g., `quest_rewards.json`) -> Server Asset Loader -> **GiveItemInteraction.CODEC** -> In-memory `GiveItemInteraction` instance -> Player Action Trigger -> `run()` method execution -> Player Inventory Component Mutation -> State synchronized to Client

---

