---
description: Architectural reference for PlayerChatEvent
---

# PlayerChatEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerChatEvent implements IAsyncEvent<String>, ICancellable {
```

## Architecture & Concepts
The PlayerChatEvent is a fundamental data-transfer object within the server's event-driven architecture. It does not represent a service or manager, but rather a transient message encapsulating the state of a single chat attempt by a player. Its primary role is to act as a mutable data carrier that flows through the server's event bus, allowing various systems and plugins to inspect, modify, or cancel the chat message before it is delivered.

Implementation of the **ICancellable** interface is the cornerstone of its design. This provides a standardized contract for systems like moderation plugins, mute managers, or custom chat channels to prevent the event's default action—sending the message—from occurring.

The **IAsyncEvent** interface signals that the event bus may process this event on a worker thread, separate from the main server tick loop. This is a critical performance consideration, preventing chat-related processing from impacting core gameplay simulation.

A key architectural choice is the inclusion of the **Formatter** interface, an implementation of the Strategy Pattern. This decouples the raw content of a message from its final, formatted presentation. Systems can replace the default formatter to inject prefixes, suffixes, color codes, or other complex formatting logic without needing to manipulate the raw string content directly.

## Lifecycle & Ownership
- **Creation:** A PlayerChatEvent is instantiated exclusively by the server's low-level network packet handler upon receiving a chat packet from a connected client. The initial state is populated with the sender, the raw message content, and a default list of targets (e.g., all players in the vicinity).

- **Scope:** The object's lifetime is extremely short, existing only for the duration of its dispatch through the event bus. It is created, passed sequentially to all registered event listeners, and is then immediately eligible for garbage collection. It holds no state that persists across server ticks.

- **Destruction:** The object is dereferenced and becomes eligible for garbage collection as soon as the event bus completes its dispatch cycle for this specific event. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The internal state of PlayerChatEvent is highly **Mutable**. This is a deliberate design choice. Event listeners are expected to modify the event's properties—such as its content, targets, or cancelled status—as a primary means of implementing chat-related features.

- **Thread Safety:** This class is **not thread-safe** and must not be treated as such. It is designed to be accessed by a single thread at a time as the event bus iterates through its listeners. The IAsyncEvent marker guarantees that the *event bus* will manage the threading context, not that the object itself can handle concurrent modifications. Storing a reference to the event and modifying it from a separate, user-managed thread will result in race conditions and undefined behavior.

## API Surface
The public API is composed almost entirely of accessors and mutators for its internal state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setContent(String content) | void | O(1) | Overwrites the raw text of the message. Central for word filters. |
| setTargets(List<PlayerRef> targets) | void | O(1) | Replaces the list of recipients. Essential for channel or private message systems. |
| setFormatter(Formatter formatter) | void | O(1) | Replaces the logic used to format the final message. Allows for custom prefixes or layouts. |
| setCancelled(boolean cancelled) | void | O(1) | Halts the event. If set to true, the message will not be sent after all listeners have run. |

## Integration Patterns

### Standard Usage
The canonical use case is to create a listener that intercepts the event, inspects its state, and applies logic. This could involve filtering content, checking permissions, or redirecting the message.

```java
// In a registered event listener class
@EventHandler
public void onPlayerChat(PlayerChatEvent event) {
    PlayerRef sender = event.getSender();
    
    // Anti-spam: Cancel event if message is too short
    if (event.getContent().length() < 2) {
        event.setCancelled(true);
        sender.sendMessage(Message.error("Your message is too short."));
        return;
    }

    // Feature: Add a "VIP" prefix for players with a specific permission
    if (sender.hasPermission("chat.vip")) {
        PlayerChatEvent.Formatter originalFormatter = event.getFormatter();
        event.setFormatter((player, message) -> {
            Message formatted = originalFormatter.format(player, message);
            // Prepend the VIP tag to the original formatted message
            return Message.translation("chat.vip_prefix").append(formatted);
        });
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new PlayerChatEvent()`. These events must originate from the server's core network layer. Manually firing a synthetic event can bypass security checks and cause client-server desynchronization.

- **Asynchronous Modification:** Do not capture the event object in a listener and attempt to modify it later on a separate thread. The event's lifecycle will have completed, and any subsequent modifications will be ignored at best or cause a ConcurrentModificationException at worst.

- **Ignoring Cancellation State:** Before performing computationally expensive operations within a listener (e.g., database lookups), always check `if (event.isCancelled())` first. This avoids wasting resources on an event that has already been nullified by a higher-priority listener.

## Data Pipeline
The flow of data for a chat message is linear and synchronous from the perspective of the event bus.

> Flow:
> Client Chat Input -> Server Network Ingress -> Packet Deserialization -> **PlayerChatEvent Instantiation** -> Event Bus Dispatch -> Registered Listeners (Inspect/Modify/Cancel) -> Final Check for Cancellation -> **Formatter Execution** -> Formatted Message sent to final Target List

