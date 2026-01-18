---
description: Architectural reference for WindowAction
---

# WindowAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Protocol Data Structure (Abstract Base)

## Definition
```java
// Signature
public abstract class WindowAction {
```

## Architecture & Concepts
The WindowAction class is an abstract base for all user-initiated actions within a game window, such as an inventory, crafting table, or chest. It serves as the cornerstone for a polymorphic data contract between the client and server, encapsulating a wide variety of interactions into a single, extensible system.

This class is not intended for direct instantiation. Instead, it defines a shared protocol for serialization and deserialization. The core architectural pattern is a static factory method, **deserialize**, which reads a numeric Type ID from the network buffer. Based on this ID, it dispatches the deserialization logic to the appropriate concrete subclass, such as CraftRecipeAction or SelectSlotAction.

This design centralizes the logic for handling a family of related network messages, ensuring that all window-related actions adhere to a consistent structure and lifecycle. It acts as the primary entry point for the network layer to decode and validate user input related to in-game UIs before passing it to the game logic for processing.

### Lifecycle & Ownership
- **Creation:** WindowAction instances are ephemeral and created under two distinct circumstances:
    1.  **Receiving (Deserialization):** The network layer invokes the static `WindowAction.deserialize` method on an incoming ByteBuf. This is the most common creation path. The method acts as a factory, returning a specific, fully-formed subclass.
    2.  **Sending (Instantiation):** Game logic on the sending side (typically the client) will instantiate a concrete subclass directly (e.g., `new SortItemsAction()`) in response to a player's input. This object is then passed to the network layer for serialization.

- **Scope:** The lifetime of a WindowAction object is extremely short. It exists only for the duration of a single network packet processing cycle. Once the corresponding packet handler has processed the action, the object is no longer referenced and becomes eligible for garbage collection.

- **Destruction:** Managed entirely by the Java Garbage Collector. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** The abstract WindowAction class is stateless. All state is contained within its concrete subclasses. These objects should be treated as **immutable Data Transfer Objects (DTOs)**. Their sole purpose is to carry a snapshot of a user's action from one system to another.

- **Thread Safety:** Instances are **not thread-safe** and must not be shared across threads. This is by design. A WindowAction is deserialized and processed on a single network thread (e.g., a Netty I/O thread). Any state changes resulting from the action must be dispatched to the main game thread via a thread-safe queue or event bus. Direct access from multiple threads will lead to race conditions and unpredictable behavior.

## API Surface
The public API is dominated by static methods for deserialization and instance methods for serialization, forming a symmetric contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static WindowAction | O(N) | Factory method. Reads a type ID and delegates to the correct subclass to construct an object from the buffer. Throws ProtocolException on unknown type ID. |
| getTypeId() | int | O(1) | Returns the unique integer identifier for the concrete subclass. |
| serializeWithTypeId(ByteBuf) | int | O(N) | Serializes the object into the provided buffer, prepending its type ID. This is the primary method for sending an action. |
| computeSizeWithTypeId() | int | O(1) | Calculates the total byte size of the serialized action, including its type ID, without performing the actual serialization. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a structural validation of the data in the buffer without full object instantiation. Used for early detection of malformed packets. |

## Integration Patterns

### Standard Usage
The primary use case is within a network packet handler. The handler receives a packet containing a window action, deserializes it, and processes the resulting object.

```java
// In a network packet handler on the server
public void handlePacket(WindowActionPacket packet) {
    ByteBuf buffer = packet.getPayload();
    
    // The static deserialize method is the correct entry point
    WindowAction action = WindowAction.deserialize(buffer, buffer.readerIndex());

    // Process the specific action
    if (action instanceof SortItemsAction) {
        gameLogic.sortPlayerInventory(this.player);
    } else if (action instanceof SelectSlotAction selectAction) {
        gameLogic.handleSlotSelection(this.player, selectAction.getSlotId());
    }
    // ... and so on
}
```

### Anti-Patterns (Do NOT do this)
- **State Modification:** Do not attempt to modify the fields of a WindowAction subclass after it has been created. Treat it as a final, immutable record of an event.
- **Cross-Thread Access:** Never pass a deserialized WindowAction instance to another thread. The network thread must complete its interaction with the object or transform it into a different, thread-safe data structure before handing off work to the main game loop.
- **Incorrect Serialization:** Do not call the `serialize` method directly. Always use `serializeWithTypeId` to ensure the polymorphic dispatcher on the receiving end can identify the action type.

## Data Pipeline
The WindowAction class is a critical component in the data flow for UI interactions, acting as the data contract between the raw byte stream and structured game logic.

> Flow (Server-Side Deserialization):
> Raw Network Packet (ByteBuf) -> **WindowAction.deserialize** -> ConcreteAction Instance (e.g., SortItemsAction) -> Packet Handler -> Game Logic Execution -> Game State Update

