---
description: Architectural reference for ChoiceBasePage
---

# ChoiceBasePage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages.choices
**Type:** Transient Template

## Definition
```java
// Signature
public abstract class ChoiceBasePage extends InteractiveCustomUIPage<ChoiceBasePage.ChoicePageEventData> {
```

## Architecture & Concepts
ChoiceBasePage is an abstract base class that serves as a server-side model for a dynamic, player-facing choice dialog. It is a foundational component within the server-driven UI framework, designed to decouple game logic from client-side UI presentation.

The core architectural pattern is a form of Model-View-Controller where ChoiceBasePage acts as the Controller and part of the Model.
1.  **Model:** The state is defined by the array of ChoiceElement objects passed during construction. Each element represents a potential choice for the player, including its requirements and resulting interactions.
2.  **View:** The view is defined by a client-side UI layout file, referenced by the pageLayout string. The server does not manage rendering; it only provides the data and structure.
3.  **Controller:** The class acts as a controller by handling the UI construction via the build method and processing player input via the handleDataEvent method.

This component orchestrates the two-way communication for a specific UI interaction. It first builds a set of commands for the client to render the UI, and then it provides the server-side logic to handle the event generated when the player makes a selection.

## Lifecycle & Ownership
-   **Creation:** A concrete subclass of ChoiceBasePage is instantiated by high-level game systems, such as a quest manager or an NPC interaction script, whenever a player must be presented with a set of options. It is created on-demand for a single interaction.
-   **Scope:** The object's lifetime is ephemeral, strictly bound to the visibility of the UI page on the client. The super constructor is called with CustomPageLifetime.CanDismiss, indicating the page is not meant to persist permanently. It exists only as long as the server needs to manage that specific choice dialog.
-   **Destruction:** The object is eligible for garbage collection once the player interaction is complete (e.g., a choice is made, or the dialog is dismissed) and all references to it are released by the server's UI management system. There is no explicit destruction method.

## Internal State & Concurrency
-   **State:** The internal state, consisting of the PlayerRef, the ChoiceElement array, and the pageLayout string, is established at construction and is treated as immutable. The class itself does not cache data or modify its configuration post-instantiation.
-   **Thread Safety:** This class is **not thread-safe** and must not be accessed concurrently. All method invocations, particularly build and handleDataEvent, are expected to be executed on the primary server thread or a dedicated, per-player thread that guarantees serialized access to player and world state. The Store and Ref parameters passed into its methods are the designated mechanisms for safe, synchronized access to the broader entity-component system.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Populates the UI builders with commands to construct the choice dialog on the client. N is the number of ChoiceElements. |
| handleDataEvent(ref, store, data) | void | O(M) | Executes game logic in response to a player's choice. M is the number of interactions for the selected element. |

## Integration Patterns

### Standard Usage
A developer must extend ChoiceBasePage to create a concrete dialog. This subclass is then instantiated with the specific choices and layout for a given interaction and passed to the player's UI manager.

```java
// 1. Define the choices and their resulting actions
ChoiceInteraction rewardInteraction = (store, ref, player) -> player.getInventory().addItem("diamond_sword");
ChoiceElement acceptQuestElement = new ChoiceElement("Accept Quest", rewardInteraction);
ChoiceElement[] elements = { acceptQuestElement };

// 2. Create an instance of a concrete subclass (e.g., QuestDialogPage)
//    This is then passed to a UI service to be displayed to the player.
QuestDialogPage questDialog = new QuestDialogPage(playerRef, elements, "layouts/quest_dialog");
playerUIManager.openPage(questDialog);
```

### Anti-Patterns (Do NOT do this)
-   **State Modification:** Do not attempt to modify the internal elements array after the object has been constructed. The page's state is intended to be immutable for the duration of its lifecycle.
-   **Instance Re-use:** Do not cache and re-use a ChoiceBasePage instance for multiple, separate UI interactions. Each time a dialog is shown to a player, a new instance must be created to ensure state integrity.
-   **Asynchronous Execution:** Do not invoke the build or handleDataEvent methods from an external, non-game-loop thread. Doing so will lead to race conditions and severe state corruption within the EntityStore.

## Data Pipeline
The flow of data for a choice interaction is a complete round-trip, beginning on the server, moving to the client for player input, and returning to the server for processing.

> Flow:
> Server Game Logic -> Instantiates **ChoiceBasePage** -> UI Manager calls build() -> UICommandBuilder generates Network Packet -> Client UI Engine -> Player Interaction -> Client sends UI Event Packet with ChoicePageEventData -> Server Network Layer decodes packet -> UI Manager routes event to **ChoiceBasePage**.handleDataEvent() -> ChoiceInteraction modifies Game State

