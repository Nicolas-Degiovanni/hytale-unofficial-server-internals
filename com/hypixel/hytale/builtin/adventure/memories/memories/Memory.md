---
description: Architectural reference for the Memory abstract class, the foundational contract for all discoverable in-game memories.
---

# Memory

**Package:** com.hypixel.hytale.builtin.adventure.memories.memories
**Type:** Abstract Contract / Data Model

## Definition
```java
// Signature
public abstract class Memory {
```

## Architecture & Concepts
The Memory class is an abstract base class that serves as the foundational contract for all "memory" types within Hytale's adventure mode. It is not a concrete, usable object but rather a schema that defines the essential properties and behaviors that any specific memory implementation must provide.

Architecturally, this class establishes a polymorphic system for handling diverse types of lore, character backstories, or significant world events that players can discover. Each concrete subclass represents a unique, static piece of game content.

The most critical architectural component is the static `CODEC` field of type `CodecMapCodec`. This indicates that the entire memory system is deeply integrated with the engine's data serialization and content loading pipeline. The engine uses this codec to deserialize memory definitions from game asset files (e.g., JSON) into their corresponding concrete Java objects at startup, mapping a unique string identifier to each `Memory` subclass.

The `equals` and `hashCode` implementations are intentionally based on `getClass()`. This is a significant design choice: it dictates that any two instances of the *same concrete memory class* are considered identical. This reinforces the concept that `Memory` objects are stateless, immutable content definitions, not unique runtime entities.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of `Memory` are not instantiated directly via a constructor in game logic. Instead, they are deserialized and instantiated by the engine's content loading system during the server or client bootstrap phase, using the master `CODEC`.
- **Scope:** `Memory` objects are content singletons. Once loaded from game data, a single instance of each concrete memory type persists for the entire application lifecycle. They act as immutable templates.
- **Destruction:** All loaded `Memory` instances are destroyed and garbage collected only when the server or client application shuts down.

## Internal State & Concurrency
- **State:** Immutable. The `Memory` class itself is stateless. All concrete implementations are expected to be immutable data carriers, representing static game content. Their methods should return constant values defined at compile-time or loaded from configuration.
- **Thread Safety:** Inherently thread-safe. Due to their immutable nature, `Memory` objects can be safely accessed and read by any thread (e.g., Game Loop, UI Thread, Network I/O) without requiring any synchronization mechanisms.

**WARNING:** Introducing mutable state into a `Memory` subclass is a severe violation of the architectural contract and will lead to unpredictable behavior and concurrency failures.

## API Surface
The public API consists entirely of abstract methods that concrete implementations must provide. This forms the data contract for what constitutes a memory.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique, machine-readable identifier for this memory. |
| getTitle() | String | O(1) | Returns the localized, human-readable title of the memory. |
| getTooltipText() | Message | O(1) | Returns the primary descriptive text, typically shown after discovery. |
| getIconPath() | String | O(1) | Returns the resource path to the UI icon representing this memory. |
| getUndiscoveredTooltipText() | Message | O(1) | Returns the placeholder text shown before the player has discovered the memory. |

## Integration Patterns

### Standard Usage
Developers do not interact with this class directly. The primary pattern is to extend it to define a new type of memory content. The engine handles the lifecycle of these objects.

```java
// 1. Define a new concrete Memory in your content module.
public class MyFirstMemory extends Memory {
    // Implement all abstract methods...
    @Override
    public String getId() { return "my_content:my_first_memory"; }
    // ...
}

// 2. Register it with the codec, typically in a static initializer block.
// This allows the engine to load it from data files.
static {
    Memory.CODEC.add("my_content:my_first_memory", MyFirstMemory.class);
}

// 3. Game logic would then query a manager for the memory by its ID.
MemoryManager manager = context.getService(MemoryManager.class);
Memory memory = manager.getMemoryById("my_content:my_first_memory");
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new MyConcreteMemory()`. The system relies on the codec-based content pipeline to manage memory instances. Direct instantiation bypasses registration and will fail.
- **Mutable Subclasses:** Do not add setters or any mutable fields to a `Memory` subclass. These objects are shared globally and must remain immutable to ensure thread safety.
- **Dynamic Registration:** Do not attempt to register new memory types with the `CODEC` after the initial application bootstrap. The content system is not designed for hot-reloading memory definitions.

## Data Pipeline
The `Memory` class is a destination for data during content loading, and a source of data for game systems during runtime.

> **Loading Pipeline:**
> Game Asset File (e.g., JSON) -> Engine File Loader -> **CodecMapCodec Deserializer** -> **Concrete Memory Instance** -> Memory Registry

> **Runtime Data Flow:**
> Game System (e.g., UI) -> Memory Registry -> **Concrete Memory Instance** -> Rendered UI Text/Icon

