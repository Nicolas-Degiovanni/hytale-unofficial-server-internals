---
description: Architectural reference for BasicCustomUIPage
---

# BasicCustomUIPage

**Package:** com.hypixel.hytale.server.core.entity.entities.player.pages
**Type:** Transient / Base Class

## Definition
```java
// Signature
public abstract class BasicCustomUIPage extends CustomUIPage {
```

## Architecture & Concepts
BasicCustomUIPage is an abstract base class that serves as a foundational component in the server-side UI framework. Its primary architectural purpose is to simplify the creation of custom user interfaces by providing a streamlined contract for developers.

This class employs the **Template Method design pattern**. It overrides the complex, multi-argument `build` method from its parent, CustomUIPage, and delegates the core UI construction logic to a new, simpler abstract method: `build(UICommandBuilder)`. This abstraction shields UI developers from needing to interact directly with lower-level engine constructs like the EntityStore or UIEventBuilder during the initial layout phase, enforcing a clear separation of concerns.

By extending BasicCustomUIPage, developers can focus exclusively on defining the structure and components of a UI, while the base class handles the boilerplate integration with the wider UI rendering and event systems.

### Lifecycle & Ownership
- **Creation:** A concrete subclass of BasicCustomUIPage is instantiated by server-side game logic whenever a specific UI needs to be presented to a player. It is never created during engine bootstrap.
- **Scope:** The lifetime of an instance is dictated by the CustomPageLifetime enum passed during construction. It can be short-lived (e.g., for a single interaction) or persist for the duration of a player's session.
- **Destruction:** The object is marked for garbage collection when the server's UIManager closes the page, either through explicit server logic, a client request, or the expiration of its defined lifetime.

## Internal State & Concurrency
- **State:** This class is stateful, inheriting a reference to the target player (PlayerRef) and its lifetime policy from CustomUIPage. This state is established at construction and is considered immutable for the object's lifetime. Subclasses should avoid introducing significant mutable state.
- **Thread Safety:** **This class is not thread-safe.** UI page instances are designed to be owned and operated by a single thread responsible for a specific player's updates. All interactions, especially calls to the `build` method, must be synchronized within the server's main game loop for that player to prevent severe data corruption and race conditions.

## API Surface
The public contract for implementors is focused entirely on the simplified build method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(commandBuilder) | abstract void | O(N) | **Template Method.** Subclasses must implement this method to construct the UI. The provided UICommandBuilder is used to add elements. Complexity is linear based on N, the number of UI components added. |

## Integration Patterns

### Standard Usage
The correct pattern is to extend this class, implement the abstract `build` method, and instantiate the subclass within game logic to be opened by the player's UIManager.

```java
// A concrete implementation for a server announcement page.
public class AnnouncementPage extends BasicCustomUIPage {

    private final String title;
    private final String message;

    public AnnouncementPage(PlayerRef playerRef, String title, String message) {
        super(playerRef, CustomPageLifetime.TEMPORARY);
        this.title = title;
        this.message = message;
    }

    @Override
    public void build(UICommandBuilder builder) {
        builder.addText(this.title, "h1");
        builder.addSeparator();
        builder.addText(this.message);
        builder.addButton("Dismiss", "event_dismiss");
    }
}

// In server game logic:
AnnouncementPage page = new AnnouncementPage(player.getPlayerRef(), "Update", "Server will restart in 5 minutes.");
player.getUIManager().openPage(page);
```

### Anti-Patterns (Do NOT do this)
- **Overriding the wrong build method:** Do not override the `build` method that accepts four arguments (Ref, UICommandBuilder, UIEventBuilder, Store). Doing so bypasses the simplification provided by this class and can break the intended UI construction flow.
- **Storing Volatile Game State:** Avoid caching mutable game state within the page class. A UI page should be a declarative representation of a UI at a point in time. Fetch required state when building the UI, do not store it long-term in instance fields.
- **Asynchronous Building:** Never call the `build` method from a separate thread. All UI operations must be performed on the main player-update thread.

## Data Pipeline
BasicCustomUIPage acts as a generator in the UI data pipeline. It does not process incoming data; it produces a structured set of UI commands that are sent to the client for rendering.

> Flow:
> Server Game Logic -> **BasicCustomUIPage.build()** -> UICommandBuilder -> Serialized UI Commands -> Network Packet -> Client UI Renderer

