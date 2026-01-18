---
description: Architectural reference for CodecStore
---

# CodecStore

**Package:** com.hypixel.hytale.codec.store
**Type:** Hierarchical Registry

## Definition
```java
// Signature
public class CodecStore {
```

## Architecture & Concepts

The CodecStore is a specialized, hierarchical key-value registry designed to manage the lifecycle and retrieval of Codec instances. Its primary role is to decouple the systems that need to serialize or deserialize data from the concrete implementations of the codecs themselves.

The core architectural pattern is a **chain of responsibility** implemented through a parent-child hierarchy. Each CodecStore instance can have a parent. When a codec is requested via getCodec, the store first searches its local cache. If the codec is not found locally, the request is delegated up the chain to its parent. This process continues until a codec is found or the top of the hierarchy is reached.

This design enables powerful scoping. A global, static CodecStore (STATIC) holds default, application-wide codecs. More specific systems, such as a network session manager or a world loader, can create their own child CodecStore instances. These child stores can register specialized or overridden codecs that are only active within that specific context, without polluting the global namespace.

A secondary but critical feature is support for lazy instantiation via `Supplier<Codec>`. This allows the system to register a factory for a codec without paying the cost of its creation until it is first requested.

**WARNING:** The result of a `Supplier.get()` call is **not** cached internally. The supplier will be invoked on every `getCodec` call for a key that is not present in the primary `codecs` map. This can lead to significant performance degradation if the supplier performs a costly operation.

## Lifecycle & Ownership

-   **Creation:** The primary `STATIC` instance is a global singleton, created upon class loading. All other instances are created manually via `new CodecStore()` or `new CodecStore(parent)`. The default constructor automatically links a new instance to the `STATIC` store as its parent, establishing the global fallback mechanism.
-   **Scope:** The `STATIC` instance is application-scoped and persists for the entire runtime. The scope of any other CodecStore is determined by its owner. For example, a store created for a client-server connection should live only as long as that connection is active.
-   **Destruction:** The `STATIC` instance is never destroyed. All other instances are subject to standard Java garbage collection and are cleaned up when they are no longer referenced by their owner.

## Internal State & Concurrency

-   **State:** The CodecStore is a mutable, stateful component. Its internal state consists of two maps: `codecs` for eagerly instantiated codecs and `codecSuppliers` for lazily-instantiated ones.
-   **Thread Safety:** This class is designed to be thread-safe for concurrent reads and writes. It achieves this by using `ConcurrentHashMap` for its internal maps. This is essential, as codecs may be registered by a main thread while being simultaneously requested by multiple network I/O threads. All public methods are safe to call from any thread without external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCodec(key) | Codec<T> | O(N) | Retrieves a codec. Complexity is O(N) where N is the depth of the parent hierarchy. Returns null if not found. |
| putCodec(key, codec) | void | O(1) | Registers an eagerly-instantiated codec in the local store. Overwrites any existing entry for the key. |
| removeCodec(key) | Codec<?> | O(1) | Removes a codec from the local store. Does not affect parent stores. |
| putCodecSupplier(key, supplier) | void | O(1) | Registers a supplier for lazy codec instantiation. |
| removeCodecSupplier(key) | Supplier<Codec<?>> | O(1) | Removes a codec supplier from the local store. |

## Integration Patterns

### Standard Usage

The most common pattern is to create a short-lived, scoped store for a specific subsystem. This isolates codec overrides and prevents unintended side effects on the global state.

```java
// 1. Create a scoped store that falls back to the global STATIC store.
CodecStore sessionStore = new CodecStore(); // Parent is STATIC by default

// 2. Register a session-specific codec. This overrides any global codec for CustomPacket.
sessionStore.putCodec(CUSTOM_PACKET_KEY, new SessionSpecificCustomPacketCodec());

// 3. During the session, use this store to get the correct codec.
// This will return the session-specific instance.
Codec<CustomPacket> codec = sessionStore.getCodec(CUSTOM_PACKET_KEY);

// 4. A request for a different codec will fall through to the STATIC store.
Codec<LoginPacket> loginCodec = sessionStore.getCodec(LOGIN_PACKET_KEY);
```

### Anti-Patterns (Do NOT do this)

-   **Global Namespace Pollution:** Avoid registering temporary or context-specific codecs directly into `CodecStore.STATIC`. This creates tight coupling and can lead to unpredictable behavior in other systems that rely on the default codecs. Always favor creating a child store.
-   **Expensive Suppliers:** Do not use `putCodecSupplier` with a supplier that performs complex, long-running, or resource-intensive operations. The supplier is invoked on *every* lookup. If a codec is expensive to create, instantiate it once and register it with `putCodec`.
-   **State-Dependent Lookups:** Do not design systems where the result of `getCodec` is expected to change for the same key on the same store instance. While you can replace codecs, treating the store as a dynamic state machine is an anti-pattern.

## Data Pipeline

The data flow for a codec lookup is a hierarchical search. The process is deterministic and follows the parent chain until a match is found or the chain is exhausted.

> Flow:
> `getCodec(key)` -> **Local `codecs` Map Lookup** -> (if miss) -> **Local `codecSuppliers` Map Lookup** -> (if miss) -> **Delegate to `parent.getCodec(key)`** -> (repeat) -> Return `Codec` or `null`

