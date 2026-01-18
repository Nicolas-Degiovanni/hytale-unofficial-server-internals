---
description: Architectural reference for CheckUniqueItemUsageInteraction
---

# CheckUniqueItemUsageInteraction

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config.server
**Type:** Transient Configuration Object

## Definition
```java
// Signature
public class CheckUniqueItemUsageInteraction extends SimpleInstantInteraction {
```

## Architecture & Concepts
The CheckUniqueItemUsageInteraction class is a specific, data-driven rule within the server's Interaction System. It is not a general-purpose service but rather a modular, single-responsibility component that represents one step in a potentially complex sequence of game logic.

Architecturally, it functions as a server-authoritative gate. Its primary role is to check a precondition—whether a player has previously used a specific unique item—before allowing a larger interaction sequence to proceed. It directly reads from and writes to an entity's component data, specifically the UniqueItemUsagesComponent, to enforce this "use-once" behavior.

The static CODEC field is the most critical architectural feature. It signifies that this class is designed to be instantiated via deserialization from a configuration file (e.g., JSON or HOCON) rather than being constructed directly in Java code. This allows game designers to attach this validation logic to any item or entity without requiring engine-level code changes, making the interaction system highly extensible.

## Lifecycle & Ownership
- **Creation:** Instances are created by the BuilderCodec system when the server loads game asset configurations. It is never instantiated using the *new* keyword in standard game logic. An instance represents a single, configured rule.
- **Scope:** An instance is stateless and effectively immutable once loaded. It persists for the entire server session, held in memory by the central interaction registry that maps configured interactions to game objects.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down or performs a full configuration reload. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It contains no mutable fields and all data it operates on is provided via the InteractionContext argument in the firstRun method. It can be considered an immutable function object.
- **Thread Safety:** The class itself is thread-safe due to its stateless nature. However, the environment in which it operates is **strictly single-threaded**. The Interaction System, including the CommandBuffer and EntityStore it accesses, must only be manipulated from the main server game-loop thread. Invoking its methods from any other thread will lead to race conditions and severe data corruption.

## API Surface
The public API is minimal, as the class is designed to be invoked by its parent framework, not by external user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWaitForDataFrom() | WaitForDataFrom | O(1) | Declares that this interaction is server-authoritative and does not wait for client data. |
| firstRun(...) | protected void | O(1) | The core logic entry point, executed by the Interaction System. Checks and updates the player's unique item usage history. |

## Integration Patterns

### Standard Usage
This class is not used directly in Java. Instead, it is specified by type within a game asset configuration file. The server's Interaction System deserializes this configuration to build the logic chain.

A hypothetical item configuration might look like this:

```json
// in "super_sword.json"
{
  "id": "hytale:super_sword",
  "interactions": [
    {
      "type": "CheckUniqueItemUsageInteraction"
    },
    {
      "type": "GrantAchievementInteraction",
      "achievementId": "hytale:used_super_sword"
    },
    {
      "type": "PlaySoundInteraction",
      "sound": "hytale:item.success"
    }
  ]
}
```

In this example, when a player interacts with the Super Sword, the system first runs CheckUniqueItemUsageInteraction. If it succeeds (returns Finished), the system proceeds to grant the achievement. If it fails, the chain is aborted.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CheckUniqueItemUsageInteraction()`. The class is designed to be managed by the configuration loader. Manual creation bypasses this system and will result in a non-functional object.
- **Manual Invocation:** Never call the `firstRun` method directly. It requires a fully-formed InteractionContext and a valid CommandBuffer, which are only guaranteed to be correct when invoked by the server's core Interaction System during a game tick.
- **Stateful Subclassing:** Subclassing this component to add state is an anti-pattern. Interactions are expected to be stateless and reusable. State should be stored in Entity Components.

## Data Pipeline
The flow of data and control for this component is linear and driven by a player action.

> Flow:
> Player Input (Network Packet) -> Server Network Handler -> Interaction System Event -> **CheckUniqueItemUsageInteraction.firstRun()** -> CommandBuffer reads UniqueItemUsagesComponent -> Logic Check -> CommandBuffer writes to component OR sends failure notification -> Interaction state set to Finished/Failed -> Interaction System proceeds or aborts chain

