---
description: Architectural reference for MetricsRegistry
---

# MetricsRegistry

**Package:** com.hypixel.hytale.metrics
**Type:** Transient / Stateful Builder

## Definition
```java
// Signature
public class MetricsRegistry<T> implements Codec<T> {
```

## Architecture & Concepts

The MetricsRegistry is a foundational component of the engine's diagnostics and telemetry system. It functions as a configurable, type-safe, and thread-safe builder for assembling complex data snapshots from a given source object. Its primary role is to decouple the *definition* of metrics from their *collection and serialization*.

Architecturally, it implements a declarative pattern. Developers register a series of named "metrics," each defined by a function that extracts a specific piece of data from a source object of generic type T. When triggered, the registry executes all registered extractor functions against a provided instance of T and serializes the results into a BSON document.

By implementing the Codec interface, a fully configured MetricsRegistry can itself be used as a nested component within other serialization systems, allowing for powerful composition of diagnostic data.

**Key Concepts:**

*   **Source Object (T):** The generic parameter T represents the root object from which all metric data is derived. This could be a game client instance, a server world, or any other high-level state object.
*   **Extractor Function:** A lambda or method reference passed to the register method. It encapsulates the logic for retrieving a single metric value from the source object T.
*   **Codec:** Each registered metric is paired with a Codec responsible for serializing the extracted value into a BsonValue. This ensures that any data type can be handled, provided a corresponding Codec exists.
*   **Encoder-Only:** This class is designed exclusively for serialization (encoding). The decode method is unsupported and will throw an exception, reinforcing its role as a data collection and reporting tool, not a data persistence format.

## Lifecycle & Ownership

*   **Creation:** A MetricsRegistry is instantiated directly via its constructor (`new MetricsRegistry()`). It is not managed by a dependency injection framework or a global registry. Typically, a high-level manager class (e.g., a ClientMetricsManager) will create and configure one or more registries during its own initialization phase.
*   **Scope:** The lifetime of a MetricsRegistry is bound to its owner. It is designed to be a long-lived object that is configured once and then used repeatedly to generate metric snapshots throughout the application's lifecycle.
*   **Destruction:** The object is subject to standard Java garbage collection. There are no native resources or explicit cleanup methods. It is reclaimed when its owner is destroyed and all references to it are released.

## Internal State & Concurrency

*   **State:** The primary internal state is the `map` field, which stores the collection of registered metrics. This state is mutable; calls to `register` modify this collection.
*   **Thread Safety:** This class is **thread-safe**. All access to the internal metric map is mediated by a StampedLock, which provides a highly efficient read-write locking mechanism.
    *   **Registration (Write Operations):** All `register` methods acquire an exclusive write lock. This ensures that the internal map cannot be read or modified by other threads while a new metric is being added.
    *   **Encoding (Read Operations):** The `encode` method acquires a shared read lock. This allows multiple threads to safely and concurrently generate metric snapshots from the same registry, as long as no registration is in progress.

**WARNING:** While the registry's internal state is thread-safe, it provides no guarantees about the thread safety of the source object T. Extractor functions provided during registration **must** be capable of safely accessing the state of T in a concurrent environment.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(id, metricsRegistry) | MetricsRegistry | O(1) | Nests another registry. The nested registry must be of type Void. |
| register(id, func, codec) | MetricsRegistry | O(1) | Registers a metric defined by an extractor function and a specific Codec. |
| register(id, func) | MetricsRegistry | O(1) | Registers a metric where the extractor function returns a MetricProvider. |
| encode(t, extraInfo) | BsonValue | O(N) | Core serialization method. Iterates N registered metrics, executes their extractors, and encodes the results. Acquires a read lock. |
| dumpToBson(t) | BsonValue | O(N) | High-level wrapper around encode for generating a BSON document. |
| dumpToJson(path, t) | void | O(N) | Generates a BSON document and writes it to the specified file path as a JSON string. |

## Integration Patterns

### Standard Usage

The intended usage follows a builder pattern. An instance is created, configured with all necessary metrics during an initialization phase, and then used to periodically dump snapshots.

```java
// 1. Create a registry for a specific source type, e.g., GameClient
MetricsRegistry<GameClient> clientRegistry = new MetricsRegistry<>();

// 2. Register metrics during initialization
clientRegistry.register("player_position", client -> client.getPlayer().getPosition(), Position.CODEC);
clientRegistry.register("fps", client -> client.getRenderer().getFps(), Codecs.INTEGER);
clientRegistry.register("network_stats", client -> client.getNetworkManager()); // NetworkManager implements MetricProvider

// 3. During gameplay, dump metrics to a file
try {
    GameClient client = ...;
    clientRegistry.dumpToJson(client);
} catch (IOException e) {
    LOGGER.error("Failed to dump client metrics", e);
}
```

### Anti-Patterns (Do NOT do this)

*   **Modification After First Use:** Do not call `register` after the registry has been passed to other threads for encoding. While the lock prevents data corruption, it can lead to inconsistent metric reports where some snapshots include the new metric and others do not. All registration should occur during a single, well-defined initialization phase.
*   **Blocking Extractor Functions:** The extractor functions are executed synchronously under a read lock during the `encode` call. A function that performs I/O, acquires other locks, or executes a long-running computation will block **all other threads** attempting to encode metrics using this registry. Extractor functions must be fast, non-blocking, and CPU-bound.
*   **Ignoring Source Object Threading:** Do not assume the source object T is safe to access. The registry only protects its own internal map. If the `GameClient` in the example above is not thread-safe, the extractor lambdas must use appropriate synchronization to read its state.

## Data Pipeline

The flow of data from a live game object to a serialized file on disk is a multi-stage process orchestrated by the MetricsRegistry.

> Flow:
> Source Object (e.g., GameClient) -> `dumpToBson()` -> **MetricsRegistry.encode()** -> [For each metric: Extractor Function -> Raw Value] -> [Codec -> BsonValue] -> Assembled BsonDocument -> JsonWriter -> JSON File on Disk

