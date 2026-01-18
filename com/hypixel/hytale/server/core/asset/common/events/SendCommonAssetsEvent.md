---
description: Architectural reference for SendCommonAssetsEvent
---

# SendCommonAssetsEvent

**Package:** com.hypixel.hytale.server.core.asset.common.events
**Type:** Transient

## Definition
```java
// Signature
public class SendCommonAssetsEvent implements IAsyncEvent<Void> {
```

## Architecture & Concepts
The SendCommonAssetsEvent is a message object used within the server's asynchronous event-driven architecture. It represents a command to deliver a specific set of common game assets to a connected client.

Its primary role is to decouple the system responsible for managing client connections and game state (the *producer*) from the I/O-bound system that loads and transmits asset data over the network (the *consumer*).

By implementing the IAsyncEvent interface, this event signals to the event bus that its handlers can and should be executed on a separate worker thread. This is a critical performance pattern, preventing the main server tick loop from blocking on network I/O operations, which can be slow and unpredictable. The generic type of Void indicates that handlers of this event are not expected to return a result; it is a "fire-and-forget" command.

## Lifecycle & Ownership
- **Creation:** Instantiated by a high-level service that manages a client's lifecycle, typically immediately after a successful handshake or login sequence. The creator is responsible for identifying which assets the client requires and providing the correct PacketHandler for that client's connection.
- **Scope:** Extremely short-lived. An instance of this event exists only for the brief period between its creation, its posting to the event bus, and the completion of its handling.
- **Destruction:** The event object holds no persistent state and is eligible for garbage collection as soon as it has been processed by all subscribed event handlers. The event bus does not retain references to completed events.

## Internal State & Concurrency
- **State:** Effectively **Immutable**. The class provides no methods to modify its internal fields (packetHandler, assets) after construction. While the fields are not declared final, the design contract is that the state is fixed upon instantiation.
- **Thread Safety:** **Conditionally Thread-Safe**. The event object itself is safe to be passed between threads, as its state is non-volatile. However, thread safety guarantees for the objects it contains, specifically the PacketHandler, are the responsibility of the underlying networking layer. Handlers for this event must assume they are operating in a multi-threaded context and interact with the PacketHandler according to its specific concurrency policies.

## API Surface
The public API is minimal, consisting only of accessors for the data payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPacketHandler() | PacketHandler | O(1) | Returns the network handler for the target client. |
| getRequestedAssets() | Asset[] | O(1) | Returns the array of assets to be sent. |

## Integration Patterns

### Standard Usage
This event is intended to be dispatched through the server's central event bus. A service identifies a need, constructs the event, and posts it without knowledge of the implementation that will handle it.

```java
// In a component like a PlayerLoginHandler
PacketHandler clientHandler = player.getPacketHandler();
Asset[] requiredAssets = assetRegistry.getCommonAssets();

// Create and dispatch the event for asynchronous processing
SendCommonAssetsEvent event = new SendCommonAssetsEvent(clientHandler, requiredAssets);
serverContext.getEventBus().post(event);

// The calling thread can now continue without waiting for assets to be sent.
```

### Anti-Patterns (Do NOT do this)
- **Synchronous Handling:** Do not create an instance of this event and pass it directly to an asset-sending service. This defeats the purpose of the asynchronous event system and risks blocking the calling thread on network I/O.
- **State Reuse:** Do not hold a reference to a SendCommonAssetsEvent object after it has been posted. It is a single-use message.
- **Payload Mutation:** Do not modify the contents of the Asset array after posting the event. The asynchronous handler will read this array from a different thread, and such modifications would lead to a race condition.

## Data Pipeline
The SendCommonAssetsEvent acts as a data-carrying message that initiates a network-bound workflow.

> Flow:
> Client Connection -> Player Login Service -> **SendCommonAssetsEvent (Creation)** -> Event Bus -> Asset Loading Service (Worker Thread) -> PacketHandler -> TCP/IP Stack -> Client

