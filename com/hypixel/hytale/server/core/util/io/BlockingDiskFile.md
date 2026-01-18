---
description: Architectural reference for BlockingDiskFile
---

# BlockingDiskFile

**Package:** com.hypixel.hytale.server.core.util.io
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class BlockingDiskFile {
```

## Architecture & Concepts
The BlockingDiskFile class is an abstract foundation for components that persist their state to a single file on disk. It employs the **Template Method Pattern**, providing a rigid, thread-safe structure for file I/O operations while delegating the specifics of data serialization and deserialization to concrete subclasses.

Its primary architectural role is to enforce a safe and consistent contract for interacting with configuration files, player data, or other critical disk resources. It abstracts away the complexities of file locking, atomic creation, and exception handling, presenting a simple `syncLoad` and `syncSave` interface. This component acts as a synchronization primitive for any system that relies on a shared file, preventing data corruption from concurrent access.

## Lifecycle & Ownership
- **Creation:** A BlockingDiskFile is never instantiated directly. A concrete subclass (e.g., a hypothetical ServerPropertiesFile) is instantiated by a manager or service that requires persistent state. For example, a ConfigurationService would create an instance during server bootstrap.
- **Scope:** The object's lifetime is tied to its owner. For server-wide configuration, the instance persists for the entire server session. For player-specific data, it might live only as long as the player is online.
- **Destruction:** The object is eligible for garbage collection when all references to it are released. It holds no native resources that require an explicit `close` or `dispose` method.

## Internal State & Concurrency
- **State:** The base class holds immutable state: the file `path`. All mutable data state is managed entirely within the concrete subclasses that extend this class.
- **Thread Safety:** This class is explicitly designed to be thread-safe and is a cornerstone of the server's file I/O concurrency model.
    - It utilizes a `ReentrantReadWriteLock` to serialize access to the underlying file.
    - **`syncLoad`:** This method acquires an **exclusive write lock**. This is a "stop-the-world" operation for this specific file, ensuring that no other threads can read from or write to the file while its contents are being loaded into memory. This guarantees a consistent in-memory state.
    - **`syncSave`:** This method acquires a **shared read lock**. This is a critical and non-obvious design choice. It allows multiple threads to trigger a save operation concurrently, as long as a `syncLoad` is not in progress. This architecture assumes that the in-memory state being saved is stable and that the subclass `write` method is a pure, read-only operation on that state.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| syncLoad() | void | O(N) | Blocks until file is loaded into memory. Acquires an exclusive lock. Creates the file via the `create` method if it does not exist. Throws RuntimeException on I/O failure. |
| syncSave() | void | O(N) | Blocks until in-memory state is written to disk. Acquires a shared lock. Throws RuntimeException on I/O failure. |
| read(BufferedReader) | abstract void | O(N) | **Subclass Responsibility.** Deserializes data from the reader and populates the object's state. |
| write(BufferedWriter) | abstract void | O(N) | **Subclass Responsibility.** Serializes the object's current state to the writer. |
| create(BufferedWriter) | abstract void | O(1) | **Subclass Responsibility.** Writes the default or initial state of the file to the writer. |

## Integration Patterns

### Standard Usage
A concrete implementation defines the data format. The server then uses this implementation to safely load and save its properties at key lifecycle points.

```java
// 1. A concrete class implements the abstract methods
public class ServerProperties extends BlockingDiskFile {
    private int maxPlayers;

    public ServerProperties(Path path) {
        super(path);
    }

    @Override
    protected void read(BufferedReader reader) throws IOException {
        // Logic to parse properties from the reader
        this.maxPlayers = Integer.parseInt(reader.readLine());
    }

    @Override
    protected void write(BufferedWriter writer) throws IOException {
        // Logic to write properties to the writer
        writer.write(String.valueOf(this.maxPlayers));
    }

    @Override
    protected void create(BufferedWriter writer) throws IOException {
        // Write default values for a new file
        writer.write("20"); // Default to 20 players
    }
}

// 2. A service uses the implementation
Path propertiesPath = server.getDataDirectory().resolve("server.properties");
ServerProperties props = new ServerProperties(propertiesPath);

// Load on startup
props.syncLoad();

// Save on shutdown or after a command changes a value
props.syncSave();
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Access:** Do not attempt to access the file managed by this class using other I/O libraries. All access must go through `syncLoad` and `syncSave` to respect the lock.
- **State Mutation in `write`:** The `write` method in a subclass must be a pure function that only reads from the object's fields. Modifying state within `write` can lead to unpredictable behavior, as it runs under a shared read lock.
- **Frequent `syncSave` Calls:** Calling `syncSave` in a high-frequency loop (e.g., per-tick) will cause severe performance degradation due to blocking disk I/O. Batch changes and save them periodically or on specific events.

## Data Pipeline

The class does not process a continuous stream of data. Instead, it performs discrete, blocking I/O operations.

> **Load Flow:**
> `syncLoad()` Call → Acquire Exclusive Write Lock → Disk Read / In-Memory Buffer → **`read()` / `create()`** → Subclass State Population → Release Lock

> **Save Flow:**
> `syncSave()` Call → Acquire Shared Read Lock → Subclass State Serialization via **`write()`** → Disk Write → Release Lock

