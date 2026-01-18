---
description: Architectural reference for DeathItemLoss
---

# DeathItemLoss

**Package:** com.hypixel.hytale.server.core.modules.entity.damage
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class DeathItemLoss {
```

## Architecture & Concepts
The DeathItemLoss class is a data-driven configuration object that encapsulates the rules for how an entity's inventory is penalized upon death. It is not a service or manager; it is a passive data structure whose state is defined externally and read by the server's core death processing logic.

Its primary architectural feature is the static **CODEC** field. This signifies that DeathItemLoss instances are designed to be deserialized from asset configuration files (e.g., JSON definitions for gameplay mechanics). This approach decouples the specific rules of item loss from the compiled game code, allowing designers to tweak death penalties without requiring a new server build.

This class serves as a parameter object for the broader death and inventory management systems. It holds the *what* (e.g., lose 50% of item durability) while the core systems implement the *how* (e.g., iterating the inventory and applying the durability change).

## Lifecycle & Ownership
- **Creation:** Instances are created through one of three paths:
    1.  **Deserialization (Primary):** The Hytale asset loading system uses the static **CODEC** to instantiate and populate objects from game configuration files. This is the standard mechanism for defining gameplay rules.
    2.  **Programmatic Instantiation:** The public constructor can be used by server-side logic or plugins to create custom death penalty rules on the fly.
    3.  **Static Factory:** The `noLossMode()` method provides a shared, immutable instance representing zero penalties, serving as a default or a null object.

- **Scope:** Transient. An instance of DeathItemLoss is typically owned by a higher-level configuration object, such as DeathConfig. Its lifetime is bound to the lifetime of its owner, which may persist for the duration of a world session. It is not intended to be a short-lived object created per-death event.

- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is garbage collected when the owning configuration object is unloaded.

## Internal State & Concurrency
- **State:** Mutable. The internal fields are not final. However, it is architecturally intended to be treated as an immutable object after its initial creation and population by the codec or constructor. Modifying its state after it has been integrated into the gameplay systems is an anti-pattern that can lead to unpredictable behavior.

- **Thread Safety:** **Not thread-safe.** The class contains no internal synchronization mechanisms. It is designed to be created and configured during a single-threaded loading phase and subsequently read by the main server thread. Concurrent reads are safe, but concurrent writes or a write concurrent with a read will produce undefined behavior. All modifications must be externally synchronized.

## API Surface
The public API consists of a constructor, a static factory, and a set of getters for reading the configured state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DeathItemLoss(...) | constructor | O(1) | Creates a new instance with specified loss parameters. |
| noLossMode() | static DeathItemLoss | O(1) | Returns a shared, static instance representing no item loss. |
| getLossMode() | DeathConfig.ItemsLossMode | O(1) | Returns the primary mode for item loss (e.g., NONE, ALL, SPECIFIC). |
| getItemsLost() | ItemStack[] | O(1) | Returns the specific array of items to be lost. Returns an empty array if null. |
| getAmountLossPercentage() | double | O(1) | Returns the percentage of an item stack's quantity to lose. |
| getDurabilityLossPercentage() | double | O(1) | Returns the percentage of an item's durability to lose. |

## Integration Patterns

### Standard Usage
This object is not typically used directly. Instead, it is retrieved from a parent configuration object within the entity death handling logic. The system then queries its properties to apply penalties.

```java
// Within a hypothetical entity death handler
void applyDeathPenalties(Entity entity) {
    // Assume deathConfig is loaded for the entity's type or game mode
    DeathItemLoss itemLossRules = deathConfig.getDeathItemLoss();

    if (itemLossRules.getLossMode() == DeathConfig.ItemsLossMode.ALL) {
        entity.getInventory().clear();
    } else {
        // ... apply percentage-based losses based on other getters
        double durabilityPenalty = itemLossRules.getDurabilityLossPercentage();
        // ... logic to reduce durability on items
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Post-Initialization Mutation:** Modifying a DeathItemLoss object after it has been loaded and is in use by the server. This breaks the assumption of configuration immutability and can cause inconsistent death penalties for different players.

    ```java
    // DANGEROUS: Modifying a shared configuration object at runtime
    DeathItemLoss rules = someSharedConfig.getDeathItemLoss();
    rules.amountLossPercentage = 1.0; // This will affect all subsequent deaths using this config
    ```

- **Direct Instantiation for Standard Cases:** Using `new DeathItemLoss()` when the `noLossMode()` static instance should be used. This creates unnecessary object allocations.

    ```java
    // BAD: Creates a new object
    DeathItemLoss noLoss = new DeathItemLoss(DeathConfig.ItemsLossMode.NONE, ItemStack.EMPTY_ARRAY, 0.0, 0.0);

    // GOOD: Uses the shared, static instance
    DeathItemLoss noLoss = DeathItemLoss.noLossMode();
    ```

## Data Pipeline
DeathItemLoss is a destination for configuration data, not a participant in a continuous data stream. Its pipeline is part of the server's asset loading process.

> Flow:
> Gameplay Config File (JSON) -> Server Asset Loader -> Hytale Codec System -> **DeathItemLoss instance** -> DeathConfig -> Entity Death Handler

