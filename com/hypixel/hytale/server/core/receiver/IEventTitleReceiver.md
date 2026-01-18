---
description: Architectural reference for IEventTitleReceiver
---

# IEventTitleReceiver

**Package:** com.hypixel.hytale.server.core.receiver
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IEventTitleReceiver {
```

## Architecture & Concepts
The IEventTitleReceiver interface defines a server-side contract for dispatching prominent, screen-covering title card events to a game client. It serves as a critical abstraction layer, decoupling server-side game logic from the client's specific UI rendering implementation.

Systems that need to communicate major game state changes—such as quest completion, zone discovery, or world event triggers—will interact with an implementation of this interface. The server's responsibility is limited to defining the content and style of the title (primary text, secondary text, icon, duration). It remains entirely unaware of how the client chooses to render this information.

This component is a fundamental part of the server's remote procedure call (RPC) mechanism for controlling high-level client UI. It ensures that game logic can command visually impactful feedback without being coupled to client-side rendering code.

### Lifecycle & Ownership
As an interface, IEventTitleReceiver itself has no lifecycle. The lifecycle described here pertains to the objects that *implement* this contract, which are typically entities representing a connected client.

- **Creation:** An object implementing this interface, such as a Player or ClientConnection, is instantiated by the server's connection manager when a client successfully authenticates and joins the game world.
- **Scope:** The implementation lives for the duration of the client's session. It is a session-scoped object.
- **Destruction:** The object is marked for garbage collection when the client disconnects or the server initiates a session termination. All references to it from game systems should be cleared to prevent memory leaks.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, any concrete implementation is expected to be stateful, as it is typically part of a larger object (like Player) that holds comprehensive state about a client's connection and status.
- **Thread Safety:** Implementations of this interface **must be thread-safe**. Server-side game logic operates across multiple threads. A quest system on one thread and a world event manager on another may attempt to show a title simultaneously. Implementations must ensure that these calls are handled gracefully, typically by serializing the commands and placing them onto a single, thread-safe network-outbound queue associated with the client's connection. Direct, concurrent modification of network buffers is a critical anti-pattern.

## API Surface
The API provides overloaded convenience methods to simplify common use cases, such as specifying default fade and hold durations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| showEventTitle(Message, Message, boolean, String, float, float, float) | void | O(1) | The canonical method. Dispatches a command to the client to display a title with explicit primary/secondary text, style, icon, and timing parameters. |
| hideEventTitle(float) | void | O(1) | Dispatches a command to the client to fade out the currently displayed event title over a specified duration. |

## Integration Patterns

### Standard Usage
The intended pattern is for a server-side system to acquire a reference to a receiver (e.g., a Player object) and invoke the method to trigger the UI event on the corresponding client.

```java
// Example: A quest system granting a reward
void completeQuest(Player player, Quest quest) {
    // Player implements IEventTitleReceiver
    Message title = Message.fromString("Quest Complete");
    Message subtitle = Message.fromString(quest.getName());

    player.showEventTitle(title, subtitle, true, "hytale:ui_icon_quest_complete");
}
```

### Anti-Patterns (Do NOT do this)
- **Implementation Casting:** Never cast an IEventTitleReceiver to a concrete class to access internal methods. This violates the abstraction and creates a brittle coupling to the server's networking internals.
- **Rapid Invocation:** Do not call showEventTitle in rapid succession (e.g., in a per-tick update loop). This will flood the client's network queue and create an unreadable, frustrating user experience. These titles are for infrequent, high-impact events.
- **Ignoring Asynchronicity:** Calling showEventTitle does not guarantee the title appears instantly. It queues a network packet. Logic should not be written with the assumption that the client has seen the title in the same server tick it was sent.

## Data Pipeline
The flow of data for an event title is unidirectional, from the server's game logic to the client's screen.

> Flow:
> Server Game Logic (e.g., Quest System) -> **IEventTitleReceiver.showEventTitle()** -> Network Packet Serialization -> TCP/IP Stack -> Client Network Receiver -> Packet Deserialization -> Client UI System -> Rendered Title on Screen

