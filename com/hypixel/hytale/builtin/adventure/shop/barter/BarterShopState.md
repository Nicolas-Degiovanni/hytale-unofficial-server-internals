---
description: Architectural reference for BarterShopState
---

# BarterShopState

**Package:** com.hypixel.hytale.builtin.adventure.shop.barter
**Type:** Singleton

## Definition
```java
// Signature
public class BarterShopState {
```

## Architecture & Concepts

The BarterShopState class is a server-side, global state manager responsible for the persistence and lifecycle of all dynamic barter shop data. It serves as the single source of truth for mutable shop properties, such as item stock levels, randomized trade offerings, and refresh timers.

Architecturally, this class decouples the static definition of a shop, defined in a BarterShopAsset, from its live, in-game state. This separation is critical for creating persistent worlds where shop inventories can change over time and survive server restarts.

The system's core responsibilities include:

*   **State Persistence:** Serializing the state of all shops to a central `barter_shop_state.json` file on disk and deserializing it on server startup. This is managed via a BSON-based Codec system.
*   **Time-Based Refresh Logic:** Managing the periodic restocking of shops. It uses the server's WorldTimeResource to calculate and trigger inventory refreshes based on intervals defined in the shop's asset.
*   **Trade Resolution:** For shops with randomized trade slots, this class manages the seed and resolution process to generate a deterministic set of trades that persist until the next refresh.
*   **Transactional Integrity:** Providing methods to safely execute trades by decrementing stock and immediately persisting the change.

It functions as a low-level service, intended to be consumed by higher-level game logic that handles direct player-to-shop interactions.

### Lifecycle & Ownership

-   **Creation:** The singleton instance is created lazily on the first call to the static `get` method. However, the system is not operational until the static `initialize` method is invoked by the server's bootstrap sequence, which provides the necessary save directory path and triggers the initial data load.
-   **Scope:** As a global singleton, BarterShopState persists for the entire server session. Its in-memory state is the canonical representation of all shop inventories.
-   **Destruction:** The instance is cleared and made eligible for garbage collection when the static `shutdown` method is called, typically as part of the server shutdown process. This method ensures a final state save to prevent data loss.

## Internal State & Concurrency

-   **State:** This class is highly mutable. Its primary internal state is the `shopStates` map, a ConcurrentHashMap that stores a ShopInstanceState object for each unique shop ID. This state is cached in memory for high-performance access and is periodically flushed to disk via the `save` method. The use of a `transient` field for `resolvedTrades` within ShopInstanceState is a key optimization to avoid serializing derived data.

-   **Thread Safety:** The class is designed to be thread-safe for high-level operations. The use of ConcurrentHashMap for the main `shopStates` collection allows for safe, concurrent reads and atomic creation of new shop states via `computeIfAbsent`.

    **WARNING:** While the top-level collection is thread-safe, individual ShopInstanceState objects are not internally synchronized. Methods like `executeTrade` perform a check-then-act on stock levels, which could theoretically create a race condition if two threads attempt to purchase the last item simultaneously. The design implicitly relies on higher-level game systems (e.g., a player's interaction queue) to serialize operations on a *single shop instance*.

## API Surface

The public API is composed entirely of static methods that operate on the singleton instance, alongside instance methods that constitute the core business logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialize(dataDirectory) | static void | O(N) | **CRITICAL.** Initializes the service, sets the save path, and loads existing state from disk. Must be called once at server startup. |
| get() | static BarterShopState | O(1) | Accesses the global singleton instance. |
| save() | static void | O(M) | Manually triggers the persistence of the current state of all shops to disk. Called automatically by most state-mutating operations. |
| shutdown() | static void | O(M) | Saves the final state and releases the singleton instance. Must be called on server shutdown. |
| getStockArray(asset, gameTime) | int[] | O(1) | Returns a copy of the stock array for a given shop. Triggers a refresh check. |
| getResolvedTrades(asset, gameTime) | BarterTrade[] | O(1) | Returns the current active trades for a shop, resolving them from a seed if necessary. Triggers a refresh check. |
| executeTrade(asset, tradeIndex, quantity, gameTime) | boolean | O(1) | Attempts to perform a trade. Decrements stock and persists state on success. Returns false if stock is insufficient. |

*N = Time to read and parse the state file from disk.*
*M = Number of active shops to serialize.*

## Integration Patterns

### Standard Usage

Interaction with BarterShopState should always be through the static `get` accessor to ensure operations are performed on the single, canonical instance. Logic should pass the relevant BarterShopAsset and the current WorldTimeResource to drive the state machine.

```java
// Example: In a server-side NPC interaction handler

// Get the global instance
BarterShopState shopState = BarterShopState.get();

// Get the current game time from the world context
Instant currentTime = world.getTime(); 

// Define the shop we are interacting with
BarterShopAsset targetShopAsset = getShopAssetForNpc(npc);

// Attempt to purchase 1 of the item at trade index 0
boolean purchaseSuccessful = shopState.executeTrade(targetShopAsset, 0, 1, currentTime);

if (purchaseSuccessful) {
    // Grant item to player
} else {
    // Inform player the item is out of stock
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new BarterShopState()`. This creates an orphaned instance that is not managed, persisted, or accessible by the rest of the server. All access must go through `BarterShopState.get()`.
-   **Skipping Initialization:** Failure to call `BarterShopState.initialize()` at server startup will result in a non-functional service. The system will operate on a transient, empty state and will be unable to save or load data, leading to a complete loss of shop persistence.
-   **Excessive Saving:** The `save` method is automatically called by critical operations like `executeTrade`. Calling it manually in a high-frequency loop will cause severe disk I/O contention and degrade server performance.
-   **State Caching:** Do not cache the results of `getStockArray` or `getResolvedTrades` for long periods. The internal state of the service can change at any time due to a time-based refresh. Always fetch the state from the service just before it is needed.

## Data Pipeline

The flow of data through this component is defined by two primary operations: state persistence and trade execution.

**State Loading (Server Startup)**
> Flow:
> `barter_shop_state.json` on Disk → BsonUtil File Read → **CODEC Deserialization** → In-Memory `BarterShopState` Instance

**Trade Execution (Gameplay)**
> Flow:
> Player Interaction → Game Logic calls `executeTrade` → **`BarterShopState` mutates stock** → `save()` is triggered → **CODEC Serialization** → BsonUtil File Write → `barter_shop_state.json` on Disk

