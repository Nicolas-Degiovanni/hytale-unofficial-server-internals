---
description: Architectural reference for InteractionPriorityCodec
---

# InteractionPriorityCodec

**Package:** com.hypixel.hytale.server.core.modules.interaction.interaction.config
**Type:** Utility

## Definition
```java
// Signature
public class InteractionPriorityCodec implements Codec<InteractionPriority> {
```

## Architecture & Concepts
The InteractionPriorityCodec is a specialized component within Hytale's data serialization framework. Its primary function is to translate the InteractionPriority object between its in-memory representation and its serialized BSON or JSON formats.

Architecturally, this codec embodies a flexible configuration pattern. It is designed to parse two distinct data structures for the same logical concept:
1.  **Simple Priority:** A single integer value, providing a straightforward default.
2.  **Complex Priority:** A map of PrioritySlot enumerations to integer values, allowing for granular, context-specific priorities (e.g., different values for MainHand versus OffHand).

This dual-format capability allows game designers and developers to specify interaction priorities with either simplicity or precision, depending on the requirements. The codec acts as the authoritative interpreter, abstracting this complexity from the rest of the game engine. It is a critical link between raw configuration files (JSON) or network data (BSON) and the strongly-typed InteractionPriority objects used by the server's interaction module.

### Lifecycle & Ownership
-   **Creation:** This codec is not intended for direct instantiation by end-users. It is typically created and managed by a central CodecRegistry or a similar factory mechanism during application bootstrap. The system registers this codec to handle all serialization and deserialization tasks for the InteractionPriority class.
-   **Scope:** As a stateless utility, an instance of InteractionPriorityCodec can be treated as a singleton. A single instance can be safely shared and reused throughout the entire server lifecycle without side effects.
-   **Destruction:** The object is subject to standard Java garbage collection. It is destroyed when the CodecRegistry that holds a reference to it is unloaded, typically during server shutdown.

## Internal State & Concurrency
-   **State:** This class is stateless. It contains no instance fields and does not cache data between calls. Its behavior is determined solely by its inputs. The internal MAP_CODEC is a static, final, and thread-safe component shared across all potential instances.
-   **Thread Safety:** The InteractionPriorityCodec is inherently thread-safe. Its methods are pure functions that do not modify any shared state. It can be invoked concurrently from multiple threads, such as network workers or world-loading threads, without requiring external synchronization or locks.

## API Surface
The public contract is defined by the Codec interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| decode(bsonValue, extraInfo) | InteractionPriority | O(N) | Deserializes a BsonValue into an InteractionPriority object. Throws CodecException if the input is not an Int32 or Document. N is the number of entries in the BSON document. |
| encode(priority, extraInfo) | BsonValue | O(N) | Serializes an InteractionPriority object into a BsonValue. Optimizes single-default-value maps into a BsonInt32. N is the number of entries in the priority map. |
| decodeJson(reader, extraInfo) | InteractionPriority | O(N) | Deserializes a raw JSON stream into an InteractionPriority object. Throws CodecException on malformed input. N is the number of key-value pairs in the JSON object. |
| toSchema(context) | Schema | O(1) | Generates a data schema describing the valid JSON/BSON structure for an InteractionPriority, used for validation and tooling. |

## Integration Patterns

### Standard Usage
Developers should never interact with this codec directly. The Hytale serialization framework uses it implicitly when processing configurations or network packets containing an InteractionPriority field. The framework's central codec registry dispatches serialization tasks to this implementation.

```java
// Hypothetical framework-level usage
// A developer would typically work with the fully decoded config object.

// 1. Framework reads a config file
String jsonConfig = "{ \"someInteraction\": { \"priority\": { \"MainHand\": 100, \"Default\": 50 } } }";

// 2. Framework uses its registry to decode the config
// The registry automatically finds and uses InteractionPriorityCodec for the "priority" field.
GameConfig config = CodecRegistry.decode(GameConfig.class, jsonConfig);

// 3. Developer uses the resulting object
InteractionPriority priority = config.getSomeInteraction().getPriority();
int mainHandPrio = priority.get(PrioritySlot.MainHand); // Returns 100
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new InteractionPriorityCodec()`. The codec system is responsible for managing codec lifecycles. Direct creation bypasses the registry and can lead to unpredictable serialization behavior.
-   **Manual Invocation:** Avoid calling the `decode` or `encode` methods directly. Rely on the higher-level, type-aware methods provided by the core serialization framework to ensure correctness and future compatibility.
-   **Assuming Input Format:** Do not write code that assumes the serialized form is always an integer or always an object. This codec is specifically designed to handle both; any consuming code should be robust to either format being produced by the `encode` method.

## Data Pipeline

**Decoding Flow (BSON/JSON to Object)**
> Raw BSON/JSON Data -> Hytale Codec Framework -> **InteractionPriorityCodec**.decode() -> In-Memory InteractionPriority Instance -> Game Logic

**Encoding Flow (Object to BSON/JSON)**
> In-Memory InteractionPriority Instance -> Hytale Codec Framework -> **InteractionPriorityCodec**.encode() -> Raw BSON/JSON Data -> Network Packet / File Storage

