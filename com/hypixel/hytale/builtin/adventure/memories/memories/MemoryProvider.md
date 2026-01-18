---
description: Architectural reference for MemoryProvider
---

# MemoryProvider

**Package:** com.hypixel.hytale.builtin.adventure.memories.memories
**Type:** Provider Base Class

## Definition
```java
// Signature
public abstract class MemoryProvider<T extends Memory> {
```

## Architecture & Concepts
The MemoryProvider is an abstract base class that establishes the fundamental contract for any system that supplies AI-perceivable "memories" to the adventure mode engine. It acts as a standardized interface for querying different categories of world state information that non-player characters (NPCs) can react to.

Each concrete implementation of MemoryProvider is responsible for a specific type of memory, such as player interactions, environmental changes, or significant world events. The class encapsulates three core concepts:
1.  **Identity:** A unique string ID to categorize the memory type (e.g., "hytale:block_broken", "hytale:chest_opened").
2.  **Serialization:** A BuilderCodec for encoding and decoding its specific memory data type, enabling persistence and network transfer.
3.  **Proximity:** A collection radius that defines how close an entity must be to perceive or "collect" a memory from this provider.

This class is a critical component of the AI sensory system, decoupling the AI's decision-making logic from the low-level details of how world state is stored and retrieved.

### Lifecycle & Ownership
-   **Creation:** Concrete subclasses of MemoryProvider are not instantiated directly by game logic. They are created and registered by the MemoriesPlugin during the server's plugin initialization phase.
-   **Scope:** An instance of a MemoryProvider persists for the entire lifecycle of the MemoriesPlugin, which is typically the full duration of a server session.
-   **Destruction:** Instances are eligible for garbage collection only when the MemoriesPlugin is unloaded, usually during server shutdown.

## Internal State & Concurrency
-   **State:** The base class itself holds immutable state. The id, codec, and defaultRadius are final and set only at construction. However, the data returned by the abstract method getAllMemories is expected to be a mutable, real-time snapshot of the game world's state.
-   **Thread Safety:** This base class is thread-safe. **WARNING:** Thread safety is NOT guaranteed for concrete implementations. The getAllMemories method is a primary point of concurrency risk. It may be invoked by asynchronous AI processing threads while memories are being generated or modified on the main server thread. Implementers of this class are responsible for ensuring that access to the underlying memory collections is properly synchronized to prevent concurrent modification exceptions and data races.

## API Surface
The public API defines the contract for retrieving memory metadata and the memories themselves.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | String | O(1) | Returns the unique identifier for this memory type. |
| getCodec() | BuilderCodec | O(1) | Returns the codec used for serializing and deserializing the memory data. |
| getCollectionRadius() | double | O(1) | Retrieves the effective radius for memory collection, dynamically sourcing it from the plugin configuration with a fallback to the default. |
| getAllMemories() | Map | O(N) | **Abstract.** Returns all active memories managed by this provider, grouped by a string key. The complexity depends heavily on the subclass implementation. |

## Integration Patterns

### Standard Usage
The primary consumer of this class is the core AI memory management system within MemoriesPlugin. The system iterates over all registered providers to aggregate a complete picture of the world for an NPC.

```java
// Example from a hypothetical AI sensory update loop
List<MemoryProvider> providers = memoriesPlugin.getRegisteredProviders();
for (MemoryProvider provider : providers) {
    Map<String, Set<Memory>> memories = provider.getAllMemories();
    // Process and integrate these memories into the AI's knowledge base
    aiKnowledge.assimilate(memories);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not instantiate concrete MemoryProvider subclasses directly. They are designed to be managed and registered exclusively by the MemoriesPlugin. Manually creating instances will result in a provider that is invisible to the core AI systems.
-   **Configuration Bypass:** Do not cache the result of getCollectionRadius. The value is designed to be dynamic and can be changed at runtime via configuration reloads. Always call the method to get the current, correct value.
-   **Unsafe Iteration:** Do not iterate over the collection returned by getAllMemories without being aware of the threading context. If calling from an asynchronous thread, assume the collection is not thread-safe unless the specific provider's documentation states otherwise.

## Data Pipeline
The MemoryProvider functions as a data source within the AI's perception pipeline. It follows a "pull" model where the AI system actively queries the provider for information.

> Flow:
> AI Behavior Tick -> MemoryManager -> **MemoryProvider.getAllMemories()** -> Live World State / Database -> AI Knowledge Base

