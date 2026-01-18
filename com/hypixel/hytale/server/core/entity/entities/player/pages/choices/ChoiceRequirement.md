---
description: Architectural reference for ChoiceRequirement
---

# ChoiceRequirement

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.choices
**Type:** Base Type / Contract

## Definition
```java
// Signature
public abstract class ChoiceRequirement {
```

## Architecture & Concepts
The ChoiceRequirement class is an abstract contract that forms the foundation of the server's data-driven conditional logic system. It is not a concrete service but a blueprint for defining specific, verifiable conditions that a player must meet to proceed with an action, such as selecting a dialogue option or accepting a quest.

Its primary architectural role is to decouple high-level game systems (like Quest or Dialogue managers) from the low-level implementation of specific rules. Instead of hardcoding checks like `if (player.getLevel() > 10)`, game designers can specify requirements in external data files.

The static `CODEC` field is the cornerstone of this design. It is a `CodecMapCodec`, a specialized deserializer that maps a string identifier (e.g., "Type": "hytale:quest_completed") from a data source to a concrete Java class that extends ChoiceRequirement. This enables polymorphic deserialization, allowing the engine to dynamically instantiate the correct requirement-checking logic at runtime based on game data. This pattern is critical for creating a moddable and easily extensible game.

All requirement evaluations are performed authoritatively on the server, ensuring game rules cannot be bypassed by a modified client.

### Lifecycle & Ownership
- **Creation:** ChoiceRequirement instances are not created directly using the `new` keyword. They are instantiated by the Hytale serialization engine via the `CODEC` when game assets (e.g., quest files, NPC interaction definitions) are loaded into memory.
- **Scope:** These objects are typically short-lived and stateless. Their lifecycle is bound to the containing data structure, such as a `DialogueNode` or `QuestObjective`. They exist only as long as their parent object is in memory and are often evaluated and then discarded.
- **Destruction:** Instances are managed by the Java Garbage Collector. There is no manual cleanup required. They are reclaimed once they are no longer referenced by any active game system.

## Internal State & Concurrency
- **State:** The abstract base class is stateless. Concrete implementations are designed to be **immutable**. Their parameters (e.g., the name of a quest to check for) are injected during deserialization and must not change during the object's lifetime. They are configuration objects, not stateful entities.
- **Thread Safety:** Implementations must be thread-safe. Given their immutable nature, this is generally guaranteed. The `canFulfillRequirement` method is expected to be a pure function that reads from the provided `EntityStore` but does not modify it or the instance's own state. The `EntityStore` itself is responsible for managing its own concurrency.

## API Surface
The public contract consists of a single abstract method that all subclasses must implement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canFulfillRequirement(Store, Ref, PlayerRef) | boolean | Varies | Evaluates the condition against the provided player and world state. Complexity depends entirely on the concrete implementation, ranging from O(1) for a simple attribute check to O(N) for an inventory scan. |

## Integration Patterns

### Standard Usage
A developer's primary interaction is not to call this class, but to extend it. A game system will then use the `CODEC` to load a list of these requirements from a configuration file and evaluate them.

```java
// 1. A concrete implementation is created
public class QuestCompletedRequirement extends ChoiceRequirement {
    private final String questId;
    // ... constructor and codec registration ...

    @Override
    public boolean canFulfillRequirement(@Nonnull Store<EntityStore> store, @Nonnull Ref<EntityStore> ref, @Nonnull PlayerRef playerRef) {
        // Logic to check if the player has completed the quest with 'questId'
        return getQuestLog(playerRef).hasCompleted(this.questId);
    }
}

// 2. A higher-level system uses it (conceptual)
List<ChoiceRequirement> requirements = loadRequirementsFromAsset("my_dialogue_choice.json");
boolean canChoose = requirements.stream().allMatch(req -> req.canFulfillRequirement(store, ref, player));

if (canChoose) {
    // Present the choice to the player
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new MyRequirement()`. The entire system is built around data-driven deserialization via the `CODEC`. Bypassing it breaks the data-driven architecture.
- **Mutable State:** Subclasses must not contain mutable fields. A requirement object may be evaluated multiple times or by multiple systems, and shared mutable state will lead to severe and difficult-to-diagnose bugs.
- **Client-Side Evaluation:** Never attempt to replicate or execute requirement logic on the client for authoritative decisions. This is a server-side contract to prevent cheating.

## Data Pipeline
The flow of data from configuration to execution is central to understanding this component's role.

> Flow:
> Game Asset (JSON/HOCON) -> Server Asset Loader -> **CodecMapCodec (deserializer)** -> In-Memory `ChoiceRequirement` instance -> Game System (e.g., DialogueManager) -> `canFulfillRequirement()` execution -> `boolean` result

