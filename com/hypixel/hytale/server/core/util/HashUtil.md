---
description: Architectural reference for HashUtil
---

# HashUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class HashUtil {
```

## Architecture & Concepts
HashUtil is a stateless, low-level cryptographic utility that provides a centralized and standardized implementation for SHA-256 hashing. It serves as a core component within the server's security and data integrity subsystems.

The primary architectural purpose of this class is to abstract the specific details of the underlying cryptographic providers, namely the Java Cryptography Architecture for the SHA-256 algorithm and Bouncy Castle for hexadecimal encoding. By funneling all hashing operations through this single utility, the system guarantees that hashes are generated consistently across all server components. This consistency is critical for operations such as password verification, file integrity checks, and generating unique content identifiers.

This class is designed to be a pure function provider; it does not maintain state and its behavior is entirely determined by its inputs.

### Lifecycle & Ownership
- **Creation:** Not applicable. As a utility class with only static methods, HashUtil is never instantiated. The Java ClassLoader loads its bytecode into memory on first access.
- **Scope:** Application-wide. The class and its static methods are available for the entire lifetime of the server application once loaded.
- **Destruction:** Not applicable. The class is unloaded from the JVM when the server process terminates or its ClassLoader is garbage collected.

## Internal State & Concurrency
- **State:** HashUtil is **stateless** and **immutable**. It does not contain any fields or cache any data. Each call to its methods operates independently, creating and destroying all necessary objects within the method's scope.
- **Thread Safety:** This class is unconditionally **thread-safe**. Its stateless nature and lack of side effects ensure that it can be safely and concurrently invoked from any number of threads without requiring external synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sha256(byte[] bytes) | String | O(N) | Computes the SHA-256 hash for the given byte array. N is the length of the input array. Throws an unchecked RuntimeException if the SHA-256 algorithm is not supported by the JVM, which indicates a fatal environment configuration error. |

## Integration Patterns

### Standard Usage
This utility should be invoked statically wherever a SHA-256 hash is required for data validation, identification, or security purposes.

```java
// Standard pattern for hashing arbitrary data
import com.hypixel.hytale.server.core.util.HashUtil;
import java.nio.charset.StandardCharsets;

byte[] fileContents = ... // Read file data
String contentHash = HashUtil.sha256(fileContents);

byte[] userData = "player-secret".getBytes(StandardCharsets.UTF_8);
String secureIdentifier = HashUtil.sha256(userData);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class is not designed to be instantiated. Do not attempt to call `new HashUtil()`. All methods must be accessed statically.
- **Redundant Hashing:** Do not implement SHA-256 hashing logic elsewhere in the codebase. Centralizing through HashUtil prevents subtle bugs arising from different hashing implementations or encoding schemes.
- **Exception Swallowing:** The `RuntimeException` thrown by `sha256` signals a catastrophic failure of the Java environment (missing a standard algorithm). It should not be caught and ignored, as this would mask a critical deployment issue. The server should be allowed to crash.

## Data Pipeline
HashUtil acts as a simple, single-stage transformation function within a larger data flow. It does not connect to other systems directly but is used as a tool by them.

> Flow:
> Raw `byte[]` -> **HashUtil.sha256** -> Hexadecimal `String` (Hash)

