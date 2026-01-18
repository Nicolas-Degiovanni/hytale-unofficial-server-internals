---
description: Architectural reference for InventoryActionType
---

# InventoryActionType

**Package:** com.hypixel.hytale.protocol
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum InventoryActionType {
   TakeAll(0),
   PutAll(1),
   QuickStack(2),
   Sort(3);
```

## Architecture & Concepts

The InventoryActionType enum is a fundamental component of the network protocol layer, responsible for serializing and deserializing player-initiated inventory operations. Its primary architectural purpose is to provide a robust, type-safe, and unambiguous contract between the client and the server for all inventory manipulations.

By mapping abstract actions like *QuickStack* to a fixed integer value, it ensures network efficiency while eliminating the use of "magic numbers" in the game logic. This prevents a class of bugs related to protocol desynchronization or incorrect integer constants.

The static factory method, fromValue, acts as a protocol validation gate. When the server or client receives a packet, this method is used to deserialize the integer back into an InventoryActionType. If the integer is out of the defined range, it throws a ProtocolException, allowing the network layer to immediately identify and reject a malformed or malicious packet.

## Lifecycle & Ownership

-   **Creation:** All instances of this enum are constants, created and initialized by the Java Virtual Machine (JVM) during class loading. Developers cannot and must not create instances manually.
-   **Scope:** Application-wide. The enum constants (TakeAll, PutAll, etc.) exist for the entire lifetime of the application process.
-   **Destruction:** Instances are managed entirely by the JVM and are garbage collected only upon application shutdown. There is no manual memory management required.

## Internal State & Concurrency

-   **State:** **Immutable**. Each enum constant holds a single, final integer field. Its state is fixed at compile time and cannot be altered during runtime.
-   **Thread Safety:** **Inherently thread-safe**. As immutable, globally accessible singletons, these constants can be safely read, passed, and compared across any number of threads without requiring locks or any other synchronization mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the action for network serialization. |
| fromValue(int value) | InventoryActionType | O(1) | Deserializes an integer from a network packet into its corresponding enum constant. Throws ProtocolException if the value is not a valid action type. |

## Integration Patterns

### Standard Usage

This enum is used whenever an inventory action needs to be sent over the network or processed by a game system. The sender uses the enum constant, and the receiver uses the fromValue method to decode it.

```java
// Example: Client sending an action to the server
InventoryActionPacket packet = new InventoryActionPacket();
packet.setAction(InventoryActionType.QuickStack);
networkManager.send(packet);

// Example: Server receiving and processing the action
int rawAction = receivedPacket.readInt();
try {
    InventoryActionType action = InventoryActionType.fromValue(rawAction);
    player.getInventory().handleAction(action);
} catch (ProtocolException e) {
    // Disconnect client for sending invalid data
    player.disconnect("Invalid inventory action: " + rawAction);
}
```

### Anti-Patterns (Do NOT do this)

-   **Using Raw Integers:** Never use the raw integer values (0, 1, 2, 3) in game logic. This defeats the entire purpose of the enum and re-introduces the risk of "magic number" bugs. Always reference the constants directly, for example, InventoryActionType.Sort.
-   **Ignoring ProtocolException:** The ProtocolException thrown by fromValue is a critical security and stability signal. It indicates a corrupted packet, a client/server version mismatch, or a malicious actor. Swallowing this exception can lead to server crashes or undefined behavior. The offending client connection should be terminated immediately.

## Data Pipeline

InventoryActionType serves as the canonical data model for inventory commands as they travel from user input to server-side execution.

> Flow:
> Client UI Interaction (e.g., Shift-Click) -> Client Input Handler -> **InventoryActionType.QuickStack** -> Network Packet Serialization (`getValue`) -> Server Packet Deserialization (`fromValue`) -> Server Inventory System Logic

