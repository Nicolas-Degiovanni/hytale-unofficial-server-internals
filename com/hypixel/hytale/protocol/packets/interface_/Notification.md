---
description: Architectural reference for Notification
---

# Notification

**Package:** com.hypixel.hytale.protocol.packets.interface_
**Type:** Data Transfer Object (DTO)

## Definition
```java
// Signature
public class Notification implements Packet {
```

## Architecture & Concepts

The Notification class is a pure Data Transfer Object representing a discrete network message. It is not a service or manager; its sole responsibility is to model the data structure for an in-game notification that can be sent over the network, typically from the server to a client. These notifications are used to display transient information to the user, such as achievement unlocks, quest updates, or system alerts.

Architecturally, it serves as a concrete implementation of the Packet interface, identifying itself with the static packet ID 212. This allows the network protocol dispatcher to route raw byte streams to the correct deserializer.

The binary serialization format is highly optimized for network efficiency:
*   **Nullable Field Bitmask:** The first byte of the payload is a bitmask named nullBits. Each bit corresponds to a nullable field (message, secondaryMessage, icon, item), indicating whether its data is present in the payload. This avoids wasting bytes for absent optional fields.
*   **Offset-Based Layout:** The packet structure consists of a fixed-size header followed by a variable-length data block. The header contains integer offsets that point to the start of each variable-sized field within the data block. This design allows fields to be read in any order and provides resilience against certain structural changes.
*   **Composition:** The Notification packet is composed of other complex serializable types, such as FormattedMessage and ItemWithAllMetadata. It delegates the serialization and deserialization of these fields to their respective implementations, promoting code reuse and modularity within the protocol.

## Lifecycle & Ownership

-   **Creation:** A Notification object is created in one of two scenarios:
    1.  **Inbound (Receiving):** Instantiated by the network protocol layer via the static `deserialize` factory method when a packet with ID 212 is received from the network buffer. The engine's packet dispatcher is the owner of this process.
    2.  **Outbound (Sending):** Instantiated directly using its constructor (`new Notification(...)`) by higher-level game logic (e.g., an achievement system) that needs to transmit a notification to a remote endpoint.

-   **Scope:** The object is transient and short-lived. It exists only for the immediate task of being serialized into a buffer or being consumed by a packet handler after deserialization. It is not designed to be stored or referenced long-term.

-   **Destruction:** The object is managed by the Java Garbage Collector. Once it has been serialized or processed by its handler, it becomes eligible for garbage collection as soon as all references to it are dropped. There are no manual destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The Notification class is a mutable container for its data. All of its primary fields are public and can be modified after instantiation. It does not cache any data or maintain any state beyond the values of its fields.

-   **Thread Safety:** **This class is not thread-safe.** It is a simple POJO with no internal locking or synchronization mechanisms. It is designed to be created, populated, and processed within a single thread, such as the main game loop or a dedicated network thread.

    **WARNING:** Modifying a Notification object from multiple threads without external synchronization will result in data corruption and undefined behavior. Do not share instances across threads after the initial write.

## API Surface

The public contract is focused on serialization, deserialization, and validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | static Notification | O(N) | Constructs a Notification object by reading from a Netty ByteBuf. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into a Netty ByteBuf according to the defined binary protocol. |
| validateStructure(buf, offset) | static ValidationResult | O(N) | Performs a read-only check of the binary data in a buffer to ensure structural integrity *before* attempting deserialization. Critical for server-side security. |
| computeSize() | int | O(N) | Calculates the total number of bytes the object will occupy when serialized. Proportional to the size of its contents. |
| getId() | int | O(1) | Returns the constant network identifier for this packet type, which is 212. |

## Integration Patterns

### Standard Usage

**Sending a Notification (Server-Side Logic)**
A system creates and populates a Notification object and hands it off to the network layer for delivery.

```java
// Example: Sending an achievement notification
FormattedMessage title = new FormattedMessage("Achievement Unlocked!");
FormattedMessage subtitle = new FormattedMessage("You crafted your first wooden sword.");

Notification achievementPacket = new Notification(
    title,
    subtitle,
    "hytale:ui/icons/achievement/crafting.png",
    null, // No item to display
    NotificationStyle.Achievement
);

// The network service handles the actual serialization and transmission
networkService.sendPacket(playerConnection, achievementPacket);
```

**Receiving a Notification (Client-Side Logic)**
Developers typically do not call deserialize directly. Instead, they register a listener or handler with a packet processing system, which invokes the handler with the fully deserialized object.

```java
// Example: A packet handler method in a UI management class
public void onNotificationReceived(Notification packet) {
    // The packet is already deserialized and ready for use
    uiManager.getNotificationView().show(packet);
}
```

### Anti-Patterns (Do NOT do this)
-   **Instance Reuse:** Do not modify and resend the same Notification object. The network layer may process serialization on a different thread or in a deferred manner. Always create a new instance for each distinct message to ensure thread safety and data integrity.
-   **Ignoring Validation:** A server must **never** trust client-sent data. Before calling deserialize on a packet received from a client, always use `validateStructure` to prevent malformed packets from causing exceptions, denial-of-service, or security vulnerabilities.
-   **Cross-Thread Access:** Do not create a Notification on one thread and pass a reference to another thread for modification without explicit synchronization. This will lead to race conditions.

## Data Pipeline

The Notification class is a key component in the flow of UI-related information between server and client.

**Outbound Flow (e.g., Server to Client)**
> Game Logic (Achievement System) -> `new Notification(...)` -> Network Service -> **`Notification.serialize()`** -> Netty Channel -> TCP/IP Stack

**Inbound Flow (e.g., Client Receiving)**
> TCP/IP Stack -> Netty Channel -> ByteBuf -> Protocol Dispatcher (reads ID 212) -> **`Notification.deserialize()`** -> Packet Handler (UI System) -> Render UI Element

