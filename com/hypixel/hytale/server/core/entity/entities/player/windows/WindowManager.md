---
description: Architectural reference for WindowManager
---

# WindowManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player.windows
**Type:** Stateful Component

## Definition
```java
// Signature
public class WindowManager {
```

## Architecture & Concepts
The WindowManager is a server-side component responsible for managing the state and lifecycle of all interactive UI windows for a single player. Each player entity possesses its own dedicated WindowManager instance, ensuring that window state is completely isolated between players.

This class acts as the authoritative source of truth for a player's open windows, such as inventories, crafting stations, and chests. Its primary function is to serve as a bridge between server-side game logic and the client's user interface. It translates high-level state changes (e.g., an item being added to a container) into the specific network packets required to render and update the UI on the client: **OpenWindow**, **UpdateWindow**, and **CloseWindow**.

The manager maintains a map of active windows, each assigned a unique, session-specific integer ID. A special-cased ID of 0 is reserved for the primary, client-requestable window, typically the player's main inventory. All other server-initiated windows receive a positive, auto-incrementing ID.

Integration with the Hytale event system is a core architectural pattern. For windows that display dynamic data, such as an **ItemContainerWindow**, the WindowManager automatically subscribes to change events from the underlying data model. When a change is detected, the corresponding window is marked as dirty, and the manager sends an **UpdateWindow** packet to the client during the next available tick.

## Lifecycle & Ownership
- **Creation:** A WindowManager is not instantiated directly. It is created and owned by a server-side Player entity as part of the player's initialization sequence. The **init** method must be called immediately after construction to link the manager to its owning **PlayerRef**, which is essential for accessing the network layer and event bus.

- **Scope:** The instance's lifetime is strictly tied to the player's server session. It persists from the moment the player joins the world until they disconnect.

- **Destruction:** The object is eligible for garbage collection when the parent Player entity is destroyed. The **closeAllWindows** method is typically invoked during the player logout process to ensure all windows are gracefully closed on the client and server, and all associated event listeners are unregistered.

## Internal State & Concurrency
- **State:** The WindowManager is highly stateful and mutable. Its core state is maintained in two concurrent hash maps:
    - **windows**: An Int2ObjectConcurrentHashMap mapping integer IDs to active **Window** instances.
    - **windowChangeEvents**: An Int2ObjectConcurrentHashMap mapping window IDs to their corresponding **EventRegistration** handles. This allows for efficient unsubscription when a window is closed.

- **Thread Safety:** This class is designed to be thread-safe for high-level operations. The use of **AtomicInteger** for ID generation and **Int2ObjectConcurrentHashMap** for state storage allows different threads (e.g., the main game thread and network processing threads) to safely open or close windows concurrently.

    **Warning:** While the collections are thread-safe, the **Window** objects themselves may not be. All operations that modify the internal state of a specific **Window** instance should be synchronized or dispatched to the main game thread to prevent data corruption and race conditions. The **updateWindows** and **validateWindows** methods, which iterate the collection, are designed to be called from a single, predictable thread, such as the main server tick.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(PlayerRef playerRef) | void | O(1) | Binds the manager to its owning player. **Critical:** Must be called before any other method. |
| openWindow(Window window) | OpenWindow | O(1) | Registers a new server-side window, assigns it an ID, and returns the corresponding network packet. The caller is responsible for sending the packet. |
| clientOpenWindow(Window window) | UpdateWindow | O(1) | Handles a client request to open a window, which always uses the reserved ID 0. Replaces any existing window at ID 0. |
| closeWindow(int id) | Window | O(1) | Closes the specified window, unregisters its event listeners, and sends a **CloseWindow** packet to the client. |
| updateWindows() | void | O(N) | Iterates through all active windows. For any window marked as dirty, an **UpdateWindow** packet is generated and sent to the client. |
| markWindowChanged(int id) | void | O(1) | An internal callback, typically triggered by the event system, to mark a window as dirty and in need of an update. |

## Integration Patterns

### Standard Usage
The typical use case involves a server-side system (e.g., block interaction logic) retrieving the player's WindowManager and using it to open a new UI. The system must handle the returned packet.

```java
// Example: Opening a crafting table window upon interaction
PlayerRef player = ...; // Obtain reference to the player
WindowManager windowManager = player.getWindowManager();

// Create the specific window type with its associated data
CraftingTableWindow craftingWindow = new CraftingTableWindow(craftingTableContainer);

// The openWindow method prepares the window and returns the packet to be sent
OpenWindow packet = windowManager.openWindow(craftingWindow);

// The caller is responsible for dispatching the packet to the client
if (packet != null) {
    player.getPacketHandler().write(packet);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using **new WindowManager()**. The component is useless until its **init** method is called with a valid **PlayerRef**. Always retrieve the managed instance from the Player entity.
- **Manual ID Management:** Do not attempt to choose or set window IDs manually via **setWindow**, except in very specific engine-level scenarios. The **openWindow** method's atomic ID generator is the only safe way to acquire a new ID.
- **State Leakage:** A WindowManager is bound to a single player. Do not cache a reference to it in a global or static context, as this will lead to incorrect behavior and state corruption.
- **Ignoring Return Values:** The **openWindow** method returns a packet that must be sent to the client. Failing to send this packet will result in a "ghost" window that exists on the server but is invisible to the player.

## Data Pipeline
The WindowManager's most complex data flow involves reacting to changes in an underlying data model, such as an inventory.

> Flow:
> External Logic (e.g., player picks up an item) -> **ItemContainer** state is modified -> **EventBus** dispatches an ItemContainerChangeEvent -> **WindowManager**'s registered listener is invoked -> **WindowManager.markWindowChanged(id)** is called -> The target **Window** is flagged as dirty -> On the next server tick, **WindowManager.updateWindows()** runs -> A new **UpdateWindow** packet is created and sent to the client -> Client UI reflects the new item.

