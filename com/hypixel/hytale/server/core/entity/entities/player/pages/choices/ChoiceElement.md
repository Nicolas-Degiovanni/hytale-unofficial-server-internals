---
description: Architectural reference for ChoiceElement
---

# ChoiceElement

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.choices
**Type:** Data Model / Abstract Base Class

## Definition
```java
// Signature
public abstract class ChoiceElement {
```

## Architecture & Concepts
The ChoiceElement class is an abstract data model that represents a single, selectable option presented to a player within a user interface. It serves as a server-side blueprint for UI elements like buttons in dialogue trees, quest selections, or other interactive menus.

Its primary architectural function is to decouple game logic from static configuration. Instead of hard-coding choices, developers define them in external data files (e.g., JSON). The Hytale server uses a powerful serialization system, centered around the static **CODEC** field, to load these definitions into memory as ChoiceElement objects at runtime.

The key architectural features are:
*   **Data-Driven Design:** Each ChoiceElement instance corresponds to a definition in a configuration file. This allows designers to create and modify player choices without changing Java code.
*   **Polymorphism:** The static **CODEC** is a CodecMapCodec, which enables the system to deserialize different concrete implementations of ChoiceElement based on a "Type" field in the source data. This allows for specialized choice types (e.g., a choice that gives an item vs. a choice that starts a quest).
*   **Condition and Action Pattern:** The class encapsulates two core concepts: **Requirements** (conditions that must be met for the choice to be available) and **Interactions** (actions that occur when the choice is selected).

This class acts as the bridge between static game data and the dynamic UI generation system. It holds all the information needed to check a player's eligibility for a choice and to construct the corresponding UI commands if they are eligible.

## Lifecycle & Ownership
- **Creation:** ChoiceElement instances are not meant to be instantiated directly in code via the *new* keyword. They are created exclusively by the server's Codec and asset loading system during the deserialization of game configuration files. The **BASE_CODEC** defines exactly how to map data from a file to the object's fields.
- **Scope:** These objects are effectively immortal singletons for the duration of the server's runtime. Once loaded from configuration, they are cached and reused for all players. Their lifecycle is tied to the server session, not to any individual player session.
- **Destruction:** Instances are cleaned up by the Java Garbage Collector only when the server is shutting down and the relevant class loaders are unloaded. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The state of a ChoiceElement is defined entirely by the configuration file it was loaded from. After deserialization, its state (displayNameKey, requirements, interactions) should be considered **deeply immutable**. The fields are protected, but there are no public mutators, and the internal arrays are not intended to be modified.
- **Thread Safety:** The class is inherently **thread-safe**. Because its state is immutable post-initialization, multiple server threads (each handling a different player) can safely invoke methods like canFulfillRequirements on the same shared ChoiceElement instance without risk of race conditions or data corruption. No locking mechanisms are required.

## API Surface
The public contract is focused on evaluating player eligibility and constructing UI components.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addButton(UICommandBuilder, UIEventBuilder, String, PlayerRef) | void | O(N) | Abstract method. Concrete implementations must populate the provided UI builders with the necessary commands and events to render this choice for the client. N is the number of interactions. |
| canFulfillRequirements(Store, Ref, PlayerRef) | boolean | O(M) | Evaluates if the specified player meets all conditions defined in the requirements array. Returns false on the first failed requirement. M is the number of requirements. |

## Integration Patterns

### Standard Usage
A higher-level game system, such as a DialogueManager, retrieves a list of configured ChoiceElement objects. It then filters these choices for a specific player and uses them to build a UI page.

```java
// Conceptual example within a hypothetical DialogueService
List<ChoiceElement> allChoices = dialogueConfig.getChoices();
UICommandBuilder commandBuilder = new UICommandBuilder();
UIEventBuilder eventBuilder = new UIEventBuilder();

for (ChoiceElement choice : allChoices) {
    // Check if the player meets the conditions for this choice
    if (choice.canFulfillRequirements(entityStore, entityRef, playerRef)) {
        // If so, add its corresponding UI button to the builders
        choice.addButton(commandBuilder, eventBuilder, "unique_choice_id", playerRef);
    }
}

// Send the constructed UI commands to the client
player.sendUIPacket(commandBuilder.build());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ConcreteChoiceElement()`. The system is designed to load choices from data files. Bypassing the codec system will lead to improperly configured objects and break the data-driven design principle.
- **State Mutation:** Do not attempt to modify the internal arrays (requirements, interactions) after the object has been loaded. This violates its immutable nature and will cause unpredictable behavior for all players interacting with that choice.
- **Caching `canFulfillRequirements`:** The result of canFulfillRequirements is specific to a player's state at a single point in time. Caching its return value is unsafe, as the player's state (e.g., inventory, quest progress) can change.

## Data Pipeline
The flow of data from configuration to a player's screen is unidirectional and centrally managed.

> Flow:
> Game Config File (JSON) -> Server Asset Loader -> **ChoiceElement.CODEC** -> In-Memory ChoiceElement Object -> Game System (e.g., DialogueManager) -> `canFulfillRequirements` Check -> `addButton` Call -> UI Command Packet -> Network Layer -> Player Client UI

