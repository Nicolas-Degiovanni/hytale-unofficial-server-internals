---
description: Architectural reference for InteractiveCustomUIPage
---

# InteractiveCustomUIPage<T>

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages
**Type:** Transient

## Definition
```java
// Signature
public abstract class InteractiveCustomUIPage<T> extends CustomUIPage {
```

## Architecture & Concepts
The InteractiveCustomUIPage is an abstract base class that serves as the server-side controller for a player's interactive user interface screen. It acts as a critical bridge between the server's network protocol layer and the core game logic, providing a structured framework for handling client-side UI events.

Its primary architectural purpose is to enforce a strongly-typed contract for UI interactions. By leveraging a generic **BuilderCodec<T>**, this class mandates that each concrete UI implementation defines a specific data structure, **T**, for its events. This pattern elevates UI event handling from raw string parsing to type-safe, object-oriented logic, significantly reducing errors and improving maintainability.

This class embodies the Template Method design pattern. It handles the complex, generic tasks of:
1.  Receiving raw event data from the network.
2.  Deserializing and validating the data into a type-safe object.
3.  Dispatching updates back to the client in the correct packet format.

The developer's responsibility is narrowed to implementing the `handleDataEvent(T data)` method, which contains the specific game logic for that UI. All operations that modify game state must be correctly scheduled onto the main world thread, a pattern enforced by the `sendUpdate` method's internal use of `world.execute`.

## Lifecycle & Ownership
-   **Creation:** An instance is created by server-side game logic when a specific UI needs to be displayed to a player. It is then registered with the target player's **PageManager**, which assumes ownership. Direct instantiation without registration is a critical error.
-   **Scope:** The object's lifetime is explicitly defined by the **CustomPageLifetime** enum provided during construction. It persists only as long as the UI is considered "open" for the player, as determined by this lifetime policy and player actions.
-   **Destruction:** The object is marked for garbage collection when the player closes the UI, its lifetime expires, or the PageManager replaces it with a different page.

## Internal State & Concurrency
-   **State:** This class is inherently stateful. It maintains a reference to the player (**PlayerRef**) and the codec for event data. Concrete subclasses are expected to hold additional state relevant to the UI they represent, such as inventory contents or crafting states.
-   **Thread Safety:** **This class is not thread-safe.** All interactions that modify world state must be marshaled to the main world thread. The `handleDataEvent` method is invoked from a network I/O thread; any attempt to directly modify entities or world data from this context will lead to severe data corruption and server instability. The provided `sendUpdate` method demonstrates the correct pattern of using `world.execute` to schedule work on the appropriate thread.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handleDataEvent(ref, store, data) | void | O(N) | **Abstract.** The primary implementation hook for subclasses. Contains the game logic to process a deserialized UI event. Complexity is dependent on the subclass implementation. |
| sendUpdate(commandBuilder, eventBuilder, clear) | void | O(1) | Schedules a UI update packet to be sent to the client. This is an asynchronous operation that queues work on the main world thread. |

## Integration Patterns

### Standard Usage
A developer must extend this class, provide a concrete type for **T**, and implement the corresponding logic.

```java
// 1. Define the data contract for UI events
public class MyUIEventData {
    public String buttonId;
    public int value;
}

// 2. Create the concrete UI Page implementation
public class MyCustomPage extends InteractiveCustomUIPage<MyUIEventData> {

    public MyCustomPage(PlayerRef playerRef) {
        // Provide the codec for the data contract
        super(playerRef, CustomPageLifetime.TEMPORARY, MyUIEventData.CODEC);
    }

    @Override
    public void handleDataEvent(Ref<EntityStore> ref, Store<EntityStore> store, MyUIEventData data) {
        // This logic is now type-safe
        if ("confirm_purchase".equals(data.buttonId)) {
            // WARNING: Schedule world-modifying logic on the main thread
            store.getExternalData().getWorld().execute(() -> {
                // ... deduct currency from player ...
            });
        }
    }
}

// 3. Open the page for a player
Player player = ...;
player.getPageManager().openCustomPage(new MyCustomPage(player.getRef()));
```

### Anti-Patterns (Do NOT do this)
-   **Direct World Modification:** Never modify entities, components, or any world state directly inside `handleDataEvent`. This method is called on a network thread and will cause race conditions. Always use `world.execute`.
-   **Blocking Operations:** Do not perform file I/O, database queries, or other long-running tasks inside `handleDataEvent`. This will block the network thread pool and severely degrade server performance.
-   **Ignoring Codec Validation:** The `BuilderCodec` performs data validation. The base class logs validation failures, but developers should not assume incoming data is always well-formed.

## Data Pipeline
The flow of data for a single client interaction is bidirectional and follows a strict, thread-aware path.

> **Inbound Flow (Client to Server):**
> Client UI Interaction → Network Packet (Raw JSON) → Server Network Layer → Player PageManager → **InteractiveCustomUIPage.handleDataEvent(String)** → BuilderCodec Deserializes JSON to Object **T** → **InteractiveCustomUIPage.handleDataEvent(T)** → Subclass Game Logic

> **Outbound Flow (Server to Client):**
> Subclass Game Logic → **sendUpdate()** → World Scheduler (`world.execute`) → Main World Thread → PageManager constructs `CustomPage` Packet → Server Network Layer → Network Packet → Client UI Renders Update

