---
description: Architectural reference for CanStartObjectiveRequirement
---

# CanStartObjectiveRequirement

**Package:** com.hypixel.hytale.builtin.adventure.objectiveshop
**Type:** Transient

## Definition
```java
// Signature
public class CanStartObjectiveRequirement extends ChoiceRequirement {
```

## Architecture & Concepts
The CanStartObjectiveRequirement class is a concrete implementation of the `ChoiceRequirement` abstraction. It serves as a data-driven rule used to determine if a player is eligible to perform an action, typically selecting an option in a UI or dialogue tree.

Its primary role is to act as a declarative gatekeeper. Instead of hard-coding logic, game designers can define this requirement in external data files (e.g., JSON or YAML). The engine's `Codec` system deserializes this data into a CanStartObjectiveRequirement instance at runtime.

This component is a bridge between the generic "Choice" system and the specific `ObjectivePlugin`. It encapsulates the logic for a single question: "Has the player met the prerequisites to begin a specific objective?". The actual stateful check is delegated to the `ObjectivePlugin`, which maintains the canonical state of all players' objective progress. This separation of data (the requirement) from logic (the objective system) is a core tenet of Hytale's data-driven and extensible architecture.

### Lifecycle & Ownership
- **Creation:** Instances are almost exclusively created by the Hytale `Codec` system during the deserialization of game content files, such as shop definitions or dialogue scripts. The static `CODEC` field defines how to map the data file's structure to the object's fields.
- **Scope:** This object is short-lived and stateless. Its lifetime is bound to the containing object, such as a `Choice` definition loaded for a specific UI interaction. It does not persist beyond the evaluation of the requirement.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup as soon as the containing `Choice` or UI page is no longer referenced. No manual destruction is necessary.

## Internal State & Concurrency
- **State:** Effectively immutable. The `objectiveId` field is set upon construction (typically by the codec) and is not modified thereafter. The class holds no other mutable state.
- **Thread Safety:** The class itself is thread-safe due to its immutability. However, the `canFulfillRequirement` method is **not** safe to call from arbitrary threads. It reads from the `EntityStore` and interacts with the `ObjectivePlugin`, both of which are designed to be accessed from the main server game loop thread. Unsynchronized access will lead to race conditions and data corruption.

## API Surface
The public contract is minimal, focusing entirely on the evaluation of the requirement.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canFulfillRequirement(store, ref, playerRef) | boolean | O(1) | Evaluates if the player can start the objective. Assumes the ObjectivePlugin uses a hash map for player state lookups. Returns false if the Player component is not found. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in procedural Java code. Its primary integration path is through data definition files. A game designer would specify this requirement as part of a larger component, such as a shop item.

**Conceptual Data Example (e.g., in a YAML file):**
```yaml
# This is a conceptual example of how this class would be defined in data.
shopItems:
  - id: "apprentice_sword"
    cost: 100
    requirements:
      - type: "CanStartObjectiveRequirement"
        objectiveId: "adventure/prologue/get_a_weapon"
```

The engine would then parse this data, instantiate a CanStartObjectiveRequirement, and automatically call `canFulfillRequirement` when a player attempts to interact with the shop item.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new CanStartObjectiveRequirement("id")` in game logic. This creates a hard-coded dependency and defeats the purpose of the data-driven architecture. All requirements should be defined in content files to allow for easy modification and iteration.
- **Complex Logic:** Do not extend this class to add complex, stateful logic. Requirements are intended to be simple, stateless data containers that delegate to larger, authoritative systems like `ObjectivePlugin`.
- **Misuse of Context:** Never call `canFulfillRequirement` without a valid `Store`, `Ref`, and `PlayerRef` provided by the server's entity processing loop. Attempting to use it in a different context will result in runtime errors or incorrect behavior.

## Data Pipeline
The class functions as a specific step in a larger data evaluation pipeline, translating a static data definition into a dynamic game logic check.

> Flow:
> Game Data File (YAML/JSON) -> Hytale Codec System -> **CanStartObjectiveRequirement Instance** -> Server Choice System -> `canFulfillRequirement` -> ObjectivePlugin Check -> Boolean Result -> UI State Update (e.g., enabling/disabling a button)

