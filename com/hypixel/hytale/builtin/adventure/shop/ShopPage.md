---
description: Architectural reference for ShopPage
---

# ShopPage

**Package:** com.hypixel.hytale.builtin.adventure.shop
**Type:** Transient

## Definition
```java
// Signature
public class ShopPage extends ChoiceBasePage {
```

## Architecture & Concepts
The ShopPage class is a specialized implementation of ChoiceBasePage, designed to render a server-defined shop interface to a player. It functions as a data-to-UI adapter, translating a high-level shop definition, identified by a string ID, into a concrete, interactive page that the player's client can display.

Its primary architectural role is to bridge the gap between the static **Asset System** and the dynamic **Player UI System**. It decouples game logic, which only needs to know a shop's ID, from the complex details of UI construction. The class fetches a pre-configured ShopAsset, extracts its UI elements, and injects them into the standardized ChoiceBasePage framework. This ensures all shops share a consistent layout and behavior, defined by the *Pages/ShopPage.ui* template, while allowing content to be defined entirely within data assets.

**WARNING:** This class is server-side only. It constructs a data model of a UI page which is then serialized and sent to the client for rendering. It does not contain any client-side rendering logic.

## Lifecycle & Ownership
-   **Creation:** A ShopPage is instantiated on-demand by server-side game logic, typically in response to a player interaction. For example, an NPC interaction script or a command handler would create a new instance when a player needs to view a shop.
-   **Scope:** The object's lifetime is strictly bound to a single viewing session by a single player. It is a transient, short-lived object.
-   **Destruction:** The instance is eligible for garbage collection as soon as the player closes the shop UI or navigates to a different page. There are no external persistent references to a ShopPage instance.

## Internal State & Concurrency
-   **State:** The ShopPage object is effectively immutable after its constructor completes. Its state is derived entirely from the ShopAsset corresponding to the provided shopId at the moment of creation. It does not cache or modify this state. All interactive state management is handled by its superclass, ChoiceBasePage.
-   **Thread Safety:** This class is **not thread-safe** and must not be shared across threads. It is designed to be created, used, and discarded within the context of a single player's update loop or event handler. The static method getShopElements accesses a global AssetMap; this underlying asset collection is assumed to be thread-safe or populated in a single-threaded environment during server initialization. Concurrent modification of the ShopAsset map during runtime is an unsupported and dangerous operation.

## API Surface
The public contract is limited to the constructor, enforcing its role as a simple data container for page initialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ShopPage(playerRef, shopId) | Constructor | O(N) | Creates a new shop page for a specific player. Complexity is proportional to the number of elements in the corresponding ShopAsset. Throws if the underlying asset system is not available. |

## Integration Patterns

### Standard Usage
A ShopPage must be instantiated with a valid PlayerRef and a pre-defined shopId. It is then typically passed to the player's page management system to be displayed.

```java
// Within a server-side event handler or script
void onPlayerInteractWithShopkeeper(PlayerRef player) {
    String shopId = "village_blacksmith_weapons";
    
    // Create a new, transient page instance for this interaction
    ShopPage shop = new ShopPage(player, shopId);

    // The player's UI manager is responsible for presenting the page
    player.getPageManager().open(shop);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Caching or Re-use:** Do not store and re-use a ShopPage instance. Each time a player opens a shop, a new instance must be created to ensure it reflects the latest asset data and initializes a clean UI state.
-   **Static Access:** Do not call the static getShopElements method directly from game logic. This method is an internal implementation detail. Bypassing the ShopPage constructor breaks the abstraction and couples your logic to the underlying asset structure.
-   **Cross-Player Usage:** Never use a ShopPage instance created for one player with a different player. The instance is tied to the PlayerRef provided during construction.

## Data Pipeline
The flow of data begins with a player action and results in a UI update on the client. The ShopPage is a critical transformation step in this server-side process.

> Flow:
> Player Interaction Event -> Game Logic Handler -> **new ShopPage(player, id)** -> ShopAsset.getAssetMap().getAsset(id) -> ChoiceBasePage(elements) -> Player Page Manager -> UI Model Serialization -> Network Packet -> Client UI Renderer

