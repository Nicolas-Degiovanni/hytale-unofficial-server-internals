---
description: Architectural reference for CraftRecipeAction
---

# CraftRecipeAction

**Package:** com.hypixel.hytale.protocol.packets.window
**Type:** Transient

## Definition
```java
// Signature
public class CraftRecipeAction extends WindowAction {
```

## Architecture & Concepts
The CraftRecipeAction class is a Data Transfer Object (DTO) that represents a specific, atomic network message within the Hytale protocol. It is not a service or a manager; its sole purpose is to encapsulate the data required for a client to request a recipe craft from the server.

As a subclass of WindowAction, it is architecturally positioned as an event related to a user interface "window," such as a crafting table or inventory screen. This class acts as the schema and serialization/deserialization engine for the `CRAFT_RECIPE` packet. It defines the precise binary layout (the "wire format") for how a recipe ID and quantity are transmitted over the network, ensuring both client and server interpret the data identically.

Its existence is purely for protocol definition. The game logic layer consumes instances of this class, but should never be tightly coupled to its serialization implementation.

## Lifecycle & Ownership
- **Creation:**
    - **Client-Side:** Instantiated by the UI input handling layer when a player confirms a crafting operation. The UI system populates the recipeId and quantity fields.
    - **Server-Side:** Instantiated exclusively by the static `deserialize` method within a Netty pipeline decoder. A new object is created for every corresponding packet received from a client.
- **Scope:** Extremely short-lived. An instance of CraftRecipeAction exists only for the duration of a single network event processing cycle. It is created, processed by a handler, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. There are no manual cleanup or `close` methods. Once the server's game logic handler has processed the action, all references to the object are dropped.

## Internal State & Concurrency
- **State:** Mutable. The fields `recipeId` and `quantity` are public and can be modified after construction. This design facilitates easy population before serialization (client) and is a direct result of deserialization (server). The object holds no other state and performs no caching.
- **Thread Safety:** **This class is not thread-safe.** It is designed to be created, populated, and processed within a single thread, typically a Netty I/O worker thread or a dedicated game logic thread.
    - **WARNING:** Sharing an instance of CraftRecipeAction across multiple threads without external synchronization will lead to race conditions and unpredictable behavior. Do not store instances of this class in shared collections or pass them between thread boundaries without proper locking.

## API Surface
The primary contract is not the instance methods, but the static serialization and validation functions that operate on network buffers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(ByteBuf, int) | static CraftRecipeAction | O(N) | Constructs a new CraftRecipeAction by reading from a Netty ByteBuf. N is the length of the recipeId string. Throws ProtocolException on malformed data. |
| serialize(ByteBuf) | int | O(N) | Writes the object's state into the provided ByteBuf. N is the length of the recipeId string. Returns the number of bytes written. |
| validateStructure(ByteBuf, int) | static ValidationResult | O(N) | Performs a read-only check on a buffer to ensure it contains a valid CraftRecipeAction structure without full deserialization. Critical for security and preventing malformed packet exploits. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will consume when serialized. Used for pre-allocating buffers. |

## Integration Patterns

### Standard Usage
This class is intended to be used by the network protocol layer. A packet handler receives a buffer, decodes it into a CraftRecipeAction object, and dispatches it to the appropriate game logic service.

```java
// Example of a server-side packet handler
// (Conceptual - actual implementation may vary)

// 1. The network layer has already identified the packet ID.
// 2. It passes the buffer to the specific deserializer.
CraftRecipeAction action = CraftRecipeAction.deserialize(packetBuffer, 0);

// 3. The fully-formed DTO is passed to the game logic system.
Player player = getPlayerFromContext();
CraftingService craftingService = context.getService(CraftingService.class);
craftingService.handleCraftRequest(player, action);
```

### Anti-Patterns (Do NOT do this)
- **State Re-use:** Do not modify a CraftRecipeAction object after it has been processed or serialized. Each action should be represented by a new instance.
- **Multi-threaded Access:** Never access a single CraftRecipeAction instance from multiple threads concurrently. The internal state is not protected by locks.
- **Ignoring Validation:** Bypassing `validateStructure` in production environments is dangerous. Always validate incoming data at the protocol boundary before attempting to deserialize to prevent denial-of-service attacks via malformed packets (e.g., a string with a huge declared length).

## Data Pipeline
The class is a single node in the client-to-server command pipeline. It translates a high-level user action into a low-level binary representation and back again.

> **Flow (Client to Server):**
> Player UI Interaction (Click Craft Button) -> UI Event Handler -> **new CraftRecipeAction("some_recipe", 1)** -> `serialize` -> Netty ByteBuf -> Network Interface -> Server -> Netty ByteBuf -> Packet Decoder -> **CraftRecipeAction.deserialize()** -> Game Logic Handler -> Inventory Update

