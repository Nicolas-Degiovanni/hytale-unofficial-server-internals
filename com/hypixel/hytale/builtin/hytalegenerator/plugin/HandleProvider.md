---
description: Architectural reference for HandleProvider
---

# HandleProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.plugin
**Type:** Transient Factory

## Definition
```java
// Signature
public class HandleProvider implements IWorldGenProvider {
```

## Architecture & Concepts
The HandleProvider class serves as a configurable factory for the primary Hytale world generator, which is an instance of the **Handle** class. It implements the **IWorldGenProvider** interface, the standard contract required by the server's world management system to obtain a concrete world generator for a new or existing world.

Its core architectural purpose is to decouple the world generation engine from server-level configuration. It encapsulates world-specific parameters, such as the world structure name and the player spawn point. When the server requests a generator, the HandleProvider uses this encapsulated state to construct a **ChunkRequest.GeneratorProfile**, which is then passed to a new **Handle** instance. This pattern ensures that the **Handle** generator itself remains stateless regarding its initial configuration, receiving all necessary parameters upon instantiation.

This class is the designated entry point for the built-in Hytale world generator, identified by the static ID *HytaleGenerator*.

## Lifecycle & Ownership
-   **Creation:** An instance of HandleProvider is created by its parent **HytaleGenerator** plugin during the plugin's bootstrap sequence. The plugin then registers this provider with the server's central world generation registry, making it available for world creation requests.
-   **Scope:** The object's lifetime is bound to the **HytaleGenerator** plugin instance. It persists as long as the plugin is enabled and registered to generate worlds. It is not a per-world or per-session object.
-   **Destruction:** The object is eligible for garbage collection when the **HytaleGenerator** plugin is disabled or the server shuts down. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
-   **State:** The HandleProvider holds mutable state consisting of the **worldStructureName** and the **playerSpawn** transform. This state acts as a template for the configuration of any new **Handle** generator it creates. To ensure state integrity, the **setPlayerSpawn** method defensively clones the incoming **Transform** object, preventing external mutations from affecting the provider's internal configuration.

-   **Thread Safety:** This class is **not thread-safe**. Its mutator methods are unsynchronized. It is architecturally expected that the provider is configured on a single thread, typically the main server thread, during world initialization.

    **WARNING:** Modifying the provider's state via **setWorldStructureName** or **setPlayerSpawn** while the world generation system is concurrently calling **getGenerator** will result in severe race conditions and unpredictable world generation outcomes. All configuration must be completed before the generator is actively used.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setWorldStructureName(String) | void | O(1) | Configures the name of the world structure preset to be used by the next generator. |
| setPlayerSpawn(Transform) | void | O(1) | Sets the default player spawn point. The input is cloned to prevent external mutation. |
| getGenerator() | IWorldGen | O(1) | Factory method. Creates and returns a new **Handle** world generator instance based on the current state. Throws **WorldGenLoadException** on failure. |

## Integration Patterns

### Standard Usage
The HandleProvider is not intended for direct use by most developers. The server's world management system retrieves it from a registry and uses it to provision a world generator. Configuration is typically performed once during server startup or in response to a world creation command.

```java
// Example of how the server might configure and use the provider
// This code is conceptual and resides within the server's core systems.

// 1. Retrieve the provider (e.g., from a registry)
IWorldGenProvider provider = worldGenRegistry.getProvider("HytaleGenerator");
HandleProvider hytaleProvider = (HandleProvider) provider;

// 2. Configure it based on world settings
hytaleProvider.setWorldStructureName("MyCustomWorldPreset");
hytaleProvider.setPlayerSpawn(new Transform(100, 200, 100));

// 3. The server then requests the generator to create the world
IWorldGen generator = hytaleProvider.getGenerator();
world.setGenerator(generator);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HandleProvider()`. The provider must be created by its parent **HytaleGenerator** plugin to ensure the critical plugin dependency is correctly injected. Failure to do so will result in a NullPointerException.
-   **Concurrent Modification:** Do not modify the provider's state from multiple threads or after the world has started generating chunks. Configuration should be treated as a "write-once" operation during the world's initialization phase.

## Data Pipeline
The HandleProvider acts as a factory at the beginning of the world generation pipeline. It does not process continuous data but rather translates a static configuration into a functional component.

> Flow:
> Server World Configuration -> **HandleProvider** (State is set) -> **getGenerator()** call -> New **Handle** (IWorldGen) instance -> World Generation System -> Chunk Generation Requests

