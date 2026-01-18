---
description: Architectural reference for PrefabSupplier
---

# PrefabSupplier

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer
**Type:** Functional Interface

## Definition
```java
// Signature
public interface PrefabSupplier extends Supplier<IPrefabBuffer> {
}
```

## Architecture & Concepts
The PrefabSupplier interface defines a contract for objects that provide instances of IPrefabBuffer. By extending the standard Java `Supplier` functional interface, it leverages a well-understood pattern for deferred creation and dependency injection.

Architecturally, this interface serves as a critical decoupling point within the server's prefab system. It separates the logic that *consumes* a prefab buffer from the logic that *creates* or *manages* one. This allows the engine to substitute different buffer allocation strategies (e.g., simple heap allocation, object pooling, or disk-backed buffers) without altering the systems that perform prefab selection and manipulation. It is a classic factory or provider pattern, specialized for the context of prefab data buffers.

## Lifecycle & Ownership
As an interface, PrefabSupplier itself has no lifecycle. The lifecycle and ownership semantics are entirely defined by the **implementing class** and the context in which it is used.

- **Creation:** An implementation of PrefabSupplier is typically instantiated and configured during server or world initialization. It is then injected into services that require prefab buffers.
- **Scope:** The scope of a PrefabSupplier instance is determined by its container. A supplier configured at the world level will persist for the lifetime of that world. A supplier used for a short-lived task will be garbage collected along with the task runner.
- **Destruction:** The interface defines no destruction or cleanup methods. Any resource management, such as closing file handles or returning objects to a pool, is the strict responsibility of the concrete implementation.

## Internal State & Concurrency
The interface is stateless. However, implementations may be stateful.

- **State:** An implementation can range from being completely stateless (e.g., creating a new buffer via `new MyBuffer()` on every call) to highly stateful (e.g., managing a pool of reusable buffer objects). Consumers of this interface **must not** make assumptions about the state of the underlying provider.
- **Thread Safety:** The contract does not enforce thread safety. It is the responsibility of the implementing class to ensure that its `get` method can be safely called from multiple threads if the use case requires it. Consumers should treat implementations as non-thread-safe unless explicitly documented otherwise.

## API Surface
The API surface is inherited directly from `java.util.function.Supplier`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | IPrefabBuffer | Varies | Supplies an instance of an IPrefabBuffer. The complexity is implementation-dependent, ranging from O(1) for a simple allocation to O(log N) for retrieval from a managed pool. |

## Integration Patterns

### Standard Usage
The primary integration pattern is dependency injection. A system that needs to operate on prefabs will be provided with a PrefabSupplier, from which it can request a buffer when needed.

```java
// A system receives a supplier via its constructor or a setter method
public class PrefabProcessor {
    private final PrefabSupplier bufferSupplier;

    public PrefabProcessor(PrefabSupplier bufferSupplier) {
        this.bufferSupplier = bufferSupplier;
    }

    public void processSelection() {
        // Obtain a buffer only when needed
        IPrefabBuffer buffer = bufferSupplier.get();
        
        // Use the buffer to perform work...
        buffer.add(somePrefab);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Fresh Instances:** Do not assume `get()` returns a new, empty buffer on every call. An implementation might return a shared or recycled instance. Always treat the returned buffer according to its specific contract.
- **Ignoring Buffer Lifecycle:** The consumer of the buffer is responsible for its lifecycle after retrieval. If the supplier is backed by a pool, failure to return the buffer after use will result in a resource leak.
- **Casting to Concrete Type:** Do not attempt to cast the supplier to a specific implementation. This violates the principle of the interface and creates a rigid, brittle coupling.

## Data Pipeline
PrefabSupplier acts as a source or factory in a data pipeline. It does not process data itself but rather provides the container for subsequent processing stages.

> Flow:
> System Logic -> **PrefabSupplier.get()** -> IPrefabBuffer Instance -> Prefab Data Population -> Downstream Processing

