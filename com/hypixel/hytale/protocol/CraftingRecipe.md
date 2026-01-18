---
description: Architectural reference for CraftingRecipe
---

# CraftingRecipe

**Package:** com.hypixel.hytale.protocol
**Type:** Transient

## Definition
```java
// Signature
public class CraftingRecipe {
```

## Architecture & Concepts
The CraftingRecipe class is a Data Transfer Object (DTO) that represents the complete definition of a crafting recipe within the Hytale protocol. Its primary role is to serve as a structured, language-native representation of recipe data that is serialized to, or deserialized from, a raw byte stream for network transmission.

This class is not a service or manager; it is a passive data structure. The architectural design is heavily optimized for network performance, prioritizing minimal payload size. This is achieved through a custom binary format composed of two distinct sections:

1.  **Fixed-Size Block (30 bytes):** This initial block contains primitive data types and, critically, a bitmask field named nullBits. This bitmask indicates which of the nullable, variable-sized fields are present in the payload. The block also contains integer offsets that point to the location of each variable-sized field's data within the subsequent variable block.

2.  **Variable-Size Block:** Following the fixed block, this section contains the actual data for complex types like strings and arrays (e.g., inputs, outputs). This separation allows for efficient parsing and skipping of data, as a reader can process the small, predictable fixed block first to understand the overall structure of the payload.

This serialization strategy is a common high-performance pattern in game development, avoiding the overhead of more verbose formats like JSON or XML.

### Lifecycle & Ownership
-   **Creation:** A CraftingRecipe instance is created under two primary circumstances:
    1.  By the network layer via the static `deserialize` method when an incoming packet containing recipe data is processed.
    2.  By server-side game logic, using the public constructor, to define a recipe that will be serialized and sent to clients.
-   **Scope:** The object is transient and has a short lifespan. It typically exists only for the duration of a single network event or game logic transaction. Once its data has been consumed (e.g., to update a UI or register the recipe in a manager), it becomes a candidate for garbage collection.
-   **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. There are no native resources or manual memory management responsibilities associated with this class.

## Internal State & Concurrency
-   **State:** The class is fully mutable. All of its fields are public, allowing for direct modification after instantiation. It encapsulates the state of a single recipe, including its identifier, inputs, outputs, and crafting requirements. It performs no caching.

-   **Thread Safety:** **This class is not thread-safe.** Its public mutable fields and lack of internal synchronization make it inherently unsafe for concurrent access.

    **WARNING:** Reading a CraftingRecipe instance on one thread while it is being written to by another will result in undefined behavior, including data corruption and runtime exceptions. All access must be externally synchronized or confined to a single thread, such as the main game loop or a dedicated network thread.

## API Surface
The public API is centered around serialization, deserialization, and validation of the network format.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| deserialize(buf, offset) | CraftingRecipe | O(N) | **[Static]** Constructs a CraftingRecipe object by reading from a ByteBuf at a given offset. Throws ProtocolException on malformed data. |
| serialize(buf) | void | O(N) | Writes the object's state into the provided ByteBuf according to the custom binary protocol. |
| validateStructure(buf, offset) | ValidationResult | O(N) | **[Static]** Performs a safe, read-only check of the binary data in a buffer to ensure it represents a valid recipe. Does not deserialize. |
| computeBytesConsumed(buf, offset) | int | O(N) | **[Static]** Calculates the total number of bytes a serialized recipe occupies in a buffer, without performing a full deserialization. |
| computeSize() | int | O(N) | Calculates the number of bytes this object will occupy when serialized. |
| clone() | CraftingRecipe | O(N) | Creates a deep copy of the recipe and its constituent arrays and objects. |

*Complexity O(N) refers to the total size of variable-length fields (strings and arrays).*

## Integration Patterns

### Standard Usage
The most common use case is for a network handler to deserialize a recipe from an incoming buffer. The `validateStructure` method should be called first on any untrusted data to prevent parsing errors or potential exploits from malformed packets.

```java
// Example of safe deserialization in a network handler
ByteBuf packetData = ...;
int recipeOffset = ...;

ValidationResult result = CraftingRecipe.validateStructure(packetData, recipeOffset);
if (!result.isValid()) {
    // Handle error, disconnect client, or log warning
    throw new ProtocolException("Invalid CraftingRecipe structure: " + result.error());
}

CraftingRecipe recipe = CraftingRecipe.deserialize(packetData, recipeOffset);
// Pass the recipe object to the game's crafting system
craftingManager.registerRecipe(recipe);
```

### Anti-Patterns (Do NOT do this)
-   **Deserializing Untrusted Data:** Never call `deserialize` directly on data from an external source without first using `validateStructure`. A malformed packet could specify invalid lengths or offsets, leading to buffer overflows or other `ProtocolException` types.
-   **Concurrent Modification:** Do not share a CraftingRecipe instance between threads. If data must be passed from a network thread to a game logic thread, either create a deep copy using `clone` or ensure a proper synchronization mechanism is in place.
-   **State Reuse:** Do not modify and re-serialize a single CraftingRecipe instance that is intended to be immutable. Its mutable design is for construction and deserialization, not for use as a long-lived, stateful object.

## Data Pipeline
The CraftingRecipe class is a critical link in the data flow between the raw network stream and the game's internal systems.

> Flow (Client Receiving a Recipe):
> Netty Channel -> ByteBuf -> **CraftingRecipe.validateStructure** -> **CraftingRecipe.deserialize** -> CraftingRecipe Instance -> CraftingManager -> UI Update / Game State Change

