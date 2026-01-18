---
description: Architectural reference for BsonUtil
---

# BsonUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class BsonUtil {
```

## Architecture & Concepts

BsonUtil is a static utility class that serves as the centralized engine for all BSON (Binary JSON) serialization, deserialization, and persistence operations within the server environment. It forms a critical bridge between in-memory data representations (BsonDocument) and external systems, primarily the file system and the network layer.

The architecture of this class is designed around safety, consistency, and performance. It abstracts the complexities of the underlying MongoDB BSON driver and Java's file I/O APIs, providing a simplified and robust interface for engine developers.

Key architectural goals include:
*   **Data Integrity:** By implementing an automatic backup-and-replace strategy for file writes (creating .bak files), it ensures that critical data like player profiles or world state is not lost due to partial writes or server crashes.
*   **Consistency:** It provides shared, pre-configured instances of BSON codecs and JSON writers. This guarantees that all BSON data serialized anywhere in the server uses the same strict formatting, indentation, and type conversion rules, preventing subtle interoperability bugs.
*   **Performance:** For file I/O, the primary API is asynchronous, returning a CompletableFuture. This prevents blocking the main server thread on slow disk operations, which is essential for maintaining server tick performance. Synchronous methods are provided but should be used with extreme caution, typically only during server startup or shutdown.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, BsonUtil is never instantiated. Its static members, including the logger and BSON codecs, are initialized by the Java ClassLoader when the class is first referenced.
-   **Scope:** The class and its static methods are available for the entire lifetime of the server application. Its scope is global.
-   **Destruction:** The class and its resources are unloaded only when the Java Virtual Machine shuts down. No manual cleanup or destruction is necessary.

## Internal State & Concurrency

-   **State:** BsonUtil is entirely stateless. It contains no mutable instance or static fields. All methods operate exclusively on the parameters provided to them. The static fields like SETTINGS and codec are immutable configurations initialized at class-load time.
-   **Thread Safety:** This class is fully thread-safe. Its stateless nature and the use of thread-safe underlying BSON components allow any method to be called from any thread without external synchronization. Asynchronous methods safely delegate I/O operations to a separate thread pool, making them ideal for use in a multi-threaded server environment.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| writeToBytes(BsonDocument) | byte[] | O(N) | Serializes a BsonDocument into a raw byte array for network transport. Returns an empty array for null input. |
| readFromBytes(byte[]) | BsonDocument | O(N) | Deserializes a byte array into a BsonDocument. Returns null for null or empty input. |
| writeDocument(Path, BsonDocument) | CompletableFuture<Void> | O(N) | Asynchronously writes a BsonDocument to a file as formatted JSON. Atomically creates a .bak file. |
| readDocument(Path) | CompletableFuture<BsonDocument> | O(N) | Asynchronously reads a BsonDocument from a file. Automatically attempts to read from the .bak file on failure. |
| readDocumentNow(Path) | BsonDocument | O(N) | **Warning:** Synchronously reads a document from a file, blocking the calling thread. Use only during server initialization. |
| toJson(BsonDocument) | String | O(N) | Converts a BsonDocument into a human-readable, indented JSON string for logging or debugging. |
| writeSync(Path, Codec, T, HytaleLogger) | void | O(N) | **Warning:** Synchronously integrates with the Hytale Codec system to encode and write an object to a file. Blocks the calling thread. |

## Integration Patterns

### Standard Usage

The primary use case is the asynchronous and safe persistence of game data. Always prefer the CompletableFuture-based methods to avoid blocking the main server thread.

```java
// Standard asynchronous file writing
BsonDocument playerData = new BsonDocument("name", new BsonString("Player1"));
playerData.put("level", new BsonInt32(10));

Path playerFile = Paths.get("server/players/player1.json");

BsonUtil.writeDocument(playerFile, playerData).exceptionally(error -> {
    LOGGER.atError().withCause(error).log("Failed to save player data for {}", playerFile);
    return null; // Handle the error
});
```

### Anti-Patterns (Do NOT do this)

-   **Blocking the Main Thread:** Never call `readDocumentNow` or `writeSync` for non-trivial files from a performance-critical thread, such as the main server game loop. Doing so will cause server lag and potentially trigger watchdog timeouts.
    ```java
    // BAD: This will freeze the server tick while the disk is accessed.
    public void onPlayerSave() {
        BsonDocument data = getPlayerData();
        BsonUtil.readDocumentNow(playerFile); // DO NOT DO THIS
    }
    ```

-   **Ignoring Failures:** The asynchronous methods return a CompletableFuture. Ignoring this return value means you will have no knowledge of whether the I/O operation succeeded or failed.
    ```java
    // BAD: If this write fails, the error is silently discarded.
    BsonUtil.writeDocument(criticalConfigFile, config);
    ```

-   **Manual Codec Instantiation:** Do not create local instances of BSON codecs or settings. This bypasses the standardized configurations in BsonUtil and can lead to inconsistent data formats.

## Data Pipeline

BsonUtil serves as a key component in the data persistence and networking pipelines.

**File Persistence Flow:**
> Game Object -> Hytale Codec System -> BsonDocument -> **BsonUtil.writeDocument** -> Formatted JSON String -> File System (.json) with .bak safety

**File Loading Flow:**
> File System (.json or .bak) -> **BsonUtil.readDocument** -> JSON String -> BsonDocument -> Hytale Codec System -> Game Object

**Network Serialization Flow:**
> BsonDocument -> **BsonUtil.writeToBytes** -> byte[] -> Netty ByteBuf -> Network Transport Layer

