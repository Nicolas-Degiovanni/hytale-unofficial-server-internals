---
description: Architectural reference for ShopElement
---

# ShopElement

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Component / Data Model

## Definition
```java
// Signature
public class ShopElement extends ChoiceElement {
```

## Architecture & Concepts
The ShopElement class is a server-side data model that represents a single, purchasable item within a user interface, such as an adventure mode shop. It is not a long-lived service but rather a transient data-driven component that bridges static game configuration with the dynamic UI system.

Its primary architectural role is to be deserialized from game data files (e.g., JSON definitions for a specific shop) via its static **CODEC**. This codec acts as a factory and validator, ensuring that game data is correctly mapped into a valid in-memory ShopElement object.

Once instantiated, a ShopElement holds all necessary data—such as cost, icon, and descriptive text—to render its corresponding UI component. The `addButton` method serves as the rendering contract, translating the object's state into a series of commands for the UI system. As a subclass of ChoiceElement, it fits into a broader framework of interactive UI elements that can be presented to a player for selection.

## Lifecycle & Ownership
- **Creation:** A ShopElement is almost exclusively instantiated by the Hytale serialization framework. The static `CODEC` field is invoked by the engine when parsing game asset files that define shop contents. Manual instantiation via its constructor is a significant anti-pattern.

- **Scope:** The lifetime of a ShopElement instance is bound to the UI context that displays it, typically a shop `Page` or a similar container. It is loaded when the shop UI is prepared for a player and exists in memory only as long as that UI is active or potentially active.

- **Destruction:** The object is eligible for garbage collection as soon as all references to its containing UI page are released. It has no explicit destruction or cleanup logic.

## Internal State & Concurrency
- **State:** The internal state (cost, iconPath) is populated once during deserialization and is considered immutable thereafter. While the fields are not declared final, modifying them post-creation is an unsupported operation that will lead to inconsistent game state.

- **Thread Safety:** This class is **not thread-safe**. It is a simple data container designed for use exclusively on the server's main game loop thread. All methods, especially those interacting with the UICommandBuilder, must be called from this thread to prevent race conditions and UI corruption.

## API Surface
The public API is minimal, focusing on UI generation and requirement validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addButton(commandBuilder, eventBuilder, selector, playerRef) | void | O(1) | Populates a UI template with this element's data (icon, name, cost). This is the primary method for rendering the element. |
| canFulfillRequirements(store, ref, playerRef) | boolean | O(1) | Determines if a player meets the conditions to interact with this element. The base implementation is a passthrough; subclasses are expected to add logic, such as checking player currency against the element's cost. |

## Integration Patterns

### Standard Usage
A developer typically interacts with a list of ShopElement objects that have been pre-loaded from game data. The standard pattern is to iterate this list to populate a UI container.

```java
// Assume 'shopPage' is a container for UI elements
// and 'elements' is a List<ShopElement> loaded from config.

UICommandBuilder commandBuilder = new UICommandBuilder();
for (ShopElement item : elements) {
    // Each call to addButton appends commands to render one item
    item.addButton(commandBuilder, eventBuilder, "#ItemList", player);
}

// Send the batch of UI commands to the player's client
player.getUI().execute(commandBuilder);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ShopElement()`. The object is not designed for manual construction; its state would be incomplete. Rely on the engine to load it from data files via its codec.
- **Post-Creation State Mutation:** Do not modify fields like `cost` or `iconPath` after the object has been loaded. This breaks the contract that the object is a faithful representation of the source game data.
- **Off-Thread UI Manipulation:** Calling `addButton` from any thread other than the main server thread is strictly forbidden and will cause unpredictable behavior or server crashes.

## Data Pipeline
The ShopElement is a critical link in the chain that transforms static configuration into an interactive player experience.

> Flow:
> Game Data File (e.g., shop.json) -> Hytale Codec Engine -> **ShopElement Instance** -> Server Game Logic (e.g., ShopPage) -> UICommandBuilder -> Network Packet -> Client UI Render

