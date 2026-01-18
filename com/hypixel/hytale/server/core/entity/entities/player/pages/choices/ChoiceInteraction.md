---
description: Architectural reference for ChoiceInteraction
---

# ChoiceInteraction

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.choices
**Type:** Base Type

## Definition
```java
// Signature
public abstract class ChoiceInteraction {
```

## Architecture & Concepts
The ChoiceInteraction class is an abstract base type that forms the core of a server-side Command Pattern. It represents a single, executable action that results from a player's choice, typically within a user interface such as a dialog box or quest journal.

This system is designed for data-driven content. Concrete implementations of ChoiceInteraction are not instantiated directly in code but are deserialized from game data files (e.g., JSON assets) at runtime. The static CODEC field, a CodecMapCodec, is the key to this architecture. It uses a "Type" identifier in the data file to map to the correct concrete Java class, allowing game designers to define complex behaviors without writing new code.

When a player performs an action, the server retrieves the corresponding ChoiceInteraction instance associated with that choice and executes its logic. The run method is the command's entry point, receiving references to the world state (EntityStore) and the acting player (PlayerRef), which it can then modify.

### Lifecycle & Ownership
-   **Creation:** Instances are created by the Hytale codec system during the asset loading phase. The server parses data files defining game content (e.g., NPC interactions), and the CODEC field orchestrates the deserialization into concrete ChoiceInteraction objects.
-   **Scope:** These objects are typically stateless templates. They are loaded into memory when their containing asset is loaded and persist as long as that asset definition is required. They are not tied to a specific player session but are shared, reusable definitions.
-   **Destruction:** Instances are eligible for garbage collection when the assets that define them are unloaded, for example, when a server shuts down or a specific world's content is purged from memory.

## Internal State & Concurrency
-   **State:** Immutable. This base class is stateless. Concrete implementations must also be stateless. All necessary context for execution is passed into the run method. This design ensures that a single deserialized instance can be safely reused for any number of players simultaneously.
-   **Thread Safety:** The objects themselves are thread-safe due to their immutability. However, the execution of the run method is **not** thread-safe. It performs mutations on the core game state (EntityStore). All calls to run must be marshaled to the main server thread responsible for the corresponding world tick to prevent race conditions and data corruption.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Store, Ref, PlayerRef) | void | Variable | Abstract method. Executes the interaction's logic against the game world. Complexity is determined by the concrete implementation. |
| CODEC | CodecMapCodec | O(1) | Static codec used by the engine to deserialize a map of choice interactions from data files. This is the primary integration point with the game's asset system. |

## Integration Patterns

### Standard Usage
Developers should extend ChoiceInteraction to define a new, reusable server-side action. This new class must then be registered with the codec system, allowing it to be referenced by name in game data files. The engine handles the lifecycle and execution.

A developer's primary task is to implement the logic within the run method.

```java
// Example of a concrete implementation
public class GiveItemInteraction extends ChoiceInteraction {
    private final Item itemToGive;
    private final int count;

    // Constructor used by the codec system
    public GiveItemInteraction(Item item, int count) {
        this.itemToGive = item;
        this.count = count;
    }

    @Override
    public void run(@Nonnull Store<EntityStore> worldStore, @Nonnull Ref<EntityStore> targetEntity, @Nonnull PlayerRef player) {
        // Logic to add an item to the player's inventory
        // This code MUST execute on the main server thread.
        player.getInventory().add(itemToGive, count);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new MyChoiceInteraction()` in game logic. The system is designed to be data-driven. All instances should be managed by the asset loading and codec systems.
-   **Stateful Implementations:** Do not add mutable fields to subclasses. This breaks the stateless, reusable template pattern and can introduce severe, hard-to-debug concurrency bugs when the same instance is used by multiple players.
-   **Asynchronous Execution:** Never call the run method from a network thread, I/O thread, or any thread other than the main world tick thread. Doing so will bypass critical state locks and lead to world corruption.

## Data Pipeline
The flow of data and execution for this component is a core server-side pattern.

> Flow:
> Game Asset (JSON) -> Server Asset Loader -> **CodecMapCodec Deserializer** -> In-Memory **ChoiceInteraction** Instance -> Player Action (Network Packet) -> Game Logic retrieves instance -> **run()** scheduled on Main Thread -> EntityStore Mutation

