---
description: Architectural reference for NPCMemoryProvider
---

# NPCMemoryProvider

**Package:** com.hypixel.hytale.builtin.adventure.memories.memories.npc
**Type:** Singleton

## Definition
```java
// Signature
public class NPCMemoryProvider extends MemoryProvider<NPCMemory> {
```

## Architecture & Concepts
The NPCMemoryProvider serves as a critical bridge between the server's core Non-Player Character (NPC) system and the Adventure Mode Memories system. Its primary function is not to store data, but to act as a dynamic data source adapter. On demand, it scans all registered NPC configurations, interprets their properties, and transforms them into a standardized `NPCMemory` format that the Memories system can consume.

This provider is responsible for dynamically discovering which NPCs are designated as "memories" for players to find. It achieves this by directly querying the central `NPCPlugin`'s `BuilderManager`, which holds the in-memory representation of all NPC assets.

A key architectural concept is the use of an `ExecutionContext` to evaluate NPC properties. Whether an NPC qualifies as a memory, its category, and its name are not determined by simple static flags. Instead, these attributes are resolved by executing logic defined within the NPC's asset data itself via the `ISpawnableWithModel` interface. This makes the system highly flexible, allowing designers to define complex conditions for memory inclusion directly in NPC configuration files.

### Lifecycle & Ownership
- **Creation:** A single instance of NPCMemoryProvider is instantiated by the `MemoriesPlugin` during the server's bootstrap sequence. It is registered as one of several potential `MemoryProvider` implementations.
- **Scope:** The object is a session-scoped singleton. It persists for the entire lifetime of the server process, from plugin initialization to server shutdown.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection when the `MemoriesPlugin` is unloaded, which typically only occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** The NPCMemoryProvider is fundamentally **stateless**. It holds no mutable instance fields and does not cache the results of its operations. Each invocation of `getAllMemories` performs a full, real-time scan and evaluation of the entire NPC registry. This ensures that the memory data is always up-to-date with the latest NPC configurations but carries significant performance implications.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread. The `getAllMemories` method iterates over collections provided by the `NPCPlugin`, which are not designed for concurrent access. Calling this method from a worker thread or asynchronous task will result in undefined behavior, likely causing a `ConcurrentModificationException` or data corruption.

## API Surface
The public contract is minimal, exposing only the functionality required by the base `MemoryProvider` contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAllMemories() | Map<String, Set<Memory>> | O(N * M) | Scans all N registered NPC builders and evaluates their memory properties with M complexity. This is a high-cost, blocking operation that can introduce performance stalls if called improperly. |

## Integration Patterns

### Standard Usage
The NPCMemoryProvider is not intended for direct use by gameplay logic developers. It is designed to be invoked once by the core `MemoriesPlugin` during its initialization phase to populate the central memory registry.

```java
// This logic resides within the core Memories system.
// It is not intended for general use.
NPCMemoryProvider provider = new NPCMemoryProvider();
Map<String, Set<Memory>> allNpcMemories = provider.getAllMemories();

// The central registry is then populated with the discovered memories.
MemoryRegistry registry = memoriesPlugin.getRegistry();
registry.registerFromProvider(allNpcMemories);
```

### Anti-Patterns (Do NOT do this)
- **Frequent Invocation:** Never call `getAllMemories` on a frequent basis, such as in a per-tick loop or in response to a player action. The method performs a full scan of all NPC configurations and is computationally expensive. It is designed to be called once at startup.
- **Direct Instantiation:** While the constructor is public, developers should not create their own instances. The provider is managed by the `MemoriesPlugin` lifecycle.
- **Asynchronous Access:** Do not invoke `getAllMemories` from a separate thread. The underlying data structures from the `NPCPlugin` are not thread-safe, and doing so will lead to server instability and data corruption.

## Data Pipeline
The NPCMemoryProvider is a key processing stage in the data pipeline that transforms static NPC asset definitions into discoverable in-game memories.

> Flow:
> NPC JSON/HOCON Assets -> `NPCPlugin.BuilderManager` -> **NPCMemoryProvider.getAllMemories()** -> `Map<String, Set<NPCMemory>>` -> `MemoriesPlugin.MemoryRegistry` -> Player Discovery System

