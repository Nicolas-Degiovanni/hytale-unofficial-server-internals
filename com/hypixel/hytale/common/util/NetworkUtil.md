---
description: Architectural reference for NetworkUtil
---

# NetworkUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class NetworkUtil {
```

## Architecture & Concepts
The NetworkUtil class is a foundational, stateless utility that provides a robust and opinionated abstraction layer over Java's low-level networking APIs, primarily the java.net package. Its core purpose is to simplify common but error-prone networking tasks required by both the client and server during initialization and diagnostics.

The system is designed to solve two primary problems:
1.  **Address Selection:** Reliably discovering a suitable, non-loopback local IP address for network services to bind to. This is critical for multiplayer functionality and service discovery. The class prioritizes IPv4 addresses over IPv6 for broader compatibility but will fall back to IPv6 if it is the only option available. The internal AddressType enum provides a flexible, predicate-based filtering mechanism to query network interfaces for specific address characteristics (e.g., multicast, site-local).

2.  **Hostname Discovery:** Determining a sensible, human-readable hostname for the local machine. This is a surprisingly complex task across different operating systems and virtualized environments (like Docker or WSL). NetworkUtil employs a sophisticated multi-stage fallback strategy, querying Java's built-in methods, system environment variables, and common filesystem locations (e.g., /etc/hostname) to find a valid name. It includes extensive filtering to reject useless or misleading names like "localhost" or raw IP literals.

This utility centralizes complex network interrogation logic, ensuring consistent behavior across the entire application and shielding higher-level components from platform-specific networking details.

## Lifecycle & Ownership
-   **Creation:** The NetworkUtil class is never instantiated. As a utility class composed entirely of static methods and fields, it is loaded into the JVM by the ClassLoader upon its first use. Its static constants (e.g., ANY_IPV4_ADDRESS) are initialized once during this class-loading phase.

-   **Scope:** The class has a global, application-wide scope. Its methods can be invoked from any thread at any time after the class has been loaded.

-   **Destruction:** The class and its static state are unloaded from memory only when the Java Virtual Machine shuts down. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** NetworkUtil is **stateless**. It does not maintain any mutable instance or static fields that change during runtime. The public static fields are constants initialized once in a static block. All methods operate on parameters passed to them or by querying the underlying operating system's network stack for real-time information.

-   **Thread Safety:** This class is **thread-safe**. All methods are re-entrant and do not modify shared state. It is safe to call any method from multiple threads concurrently without external synchronization.

    **Warning:** While the class itself is thread-safe, methods like getFirstNonLoopbackAddress and getHostName perform I/O operations by querying the operating system's network interfaces. These operations can have performance implications and should not be called in performance-critical, tight loops.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFirstNonLoopbackAddress() | InetAddress | Variable (I/O) | Scans all network interfaces to find the first usable, non-local address. Prioritizes IPv4. Returns null if none is found. Throws SocketException on OS-level errors. |
| getFirstAddressWith(AddressType... include) | InetAddress | Variable (I/O) | A generalized version of the above, allowing for custom filtering based on address properties defined in the AddressType enum. |
| getHostName() | String | Variable (I/O) | Executes a multi-stage search for a valid local hostname, involving network lookups, environment variable checks, and filesystem reads. Can be slow. |
| toSocketString(InetSocketAddress address) | String | O(L) | Formats a socket address into a string, correctly handling the bracket notation required for IPv6 addresses (e.g., [::1]:8080). |
| addressMatchesAll(InetAddress, AddressType...) | boolean | O(N) | Checks if a given address satisfies all specified address type predicates. |

## Integration Patterns

### Standard Usage
NetworkUtil is typically used during the bootstrap phase of a network-aware application, such as a game server, to determine which IP address to listen on.

```java
// How a developer should normally use this
try {
    InetAddress bindAddress = NetworkUtil.getFirstNonLoopbackAddress();
    if (bindAddress != null) {
        // Proceed to bind a server socket to this address
        ServerSocket server = new ServerSocket(port, 50, bindAddress);
    } else {
        // Handle the case where no suitable network interface is available
        Log.warn("Could not find a non-loopback address to bind to.");
    }
} catch (SocketException e) {
    Log.error("Failed to query network interfaces.", e);
}
```

### Anti-Patterns (Do NOT do this)
-   **Calling in a Game Loop:** Do not call getHostName or getFirstNonLoopbackAddress repeatedly in a performance-sensitive loop. These methods query the OS and can incur I/O latency. Cache the result at startup if it is needed frequently.

-   **Ignoring Null Returns:** Methods that search for addresses can return null if no suitable interface is found (e.g., the machine is offline). Always perform a null check on the result to avoid a NullPointerException.

-   **Assuming Hostname is an Address:** The string returned by getHostName is a name, not necessarily a resolvable IP address literal. Do not attempt to parse it as an IP without proper resolution.

## Data Pipeline
NetworkUtil acts as a data source, not a processing stage. It queries the host operating system for network configuration details and provides them to the application.

> Flow:
> Host OS Network Stack -> java.net.NetworkInterface API -> **NetworkUtil** -> Application Bootstrap Logic (e.g., Server Socket Binding)

