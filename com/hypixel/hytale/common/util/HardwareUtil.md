---
description: Architectural reference for HardwareUtil
---

# HardwareUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class HardwareUtil {
```

## Architecture & Concepts
The HardwareUtil class is a stateless, OS-aware utility responsible for providing a single, stable hardware identifier (UUID) for the machine executing the application. Its primary function is to support systems that require a persistent machine fingerprint, such as analytics, telemetry, and certain anti-cheat measures.

The core architectural pattern is a multi-layered fallback strategy, tailored for each major operating system (Windows, macOS, Linux). For a given OS, the system attempts a sequence of methods to retrieve an ID, starting with the most reliable and canonical source. If a method fails—due to permissions, missing tools, or unexpected output—it proceeds to the next in the sequence. This resilience ensures the highest possible success rate in diverse user environments.

A critical design choice is the use of a caching supplier, implemented via SupplierUtil.cache. The process of querying hardware information involves executing external commands or reading system files, which are computationally expensive and I/O-bound operations. To prevent performance degradation, the hardware UUID is computed **only once** upon the first request. All subsequent calls to getUUID within the same application session receive the cached result instantly.

## Lifecycle & Ownership
- **Creation:** As a static utility class, HardwareUtil is never instantiated. Its static members, including the cached suppliers, are initialized by the JVM ClassLoader when the class is first referenced in code.
- **Scope:** The class and its cached state are scoped to the application's lifetime. The computed hardware UUID, once retrieved, persists in memory until the JVM process terminates.
- **Destruction:** All resources are reclaimed by the JVM upon application shutdown. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The class itself is stateless. However, it manages immutable state via three private, static Supplier fields: WINDOWS, MAC, and LINUX. Each supplier, when first invoked, computes and caches the UUID for its respective platform. Once computed, this state does not change for the duration of the application session.
- **Thread Safety:** The class is fully thread-safe. The public API consists of a single static method, getUUID. The underlying caching mechanism provided by SupplierUtil is assumed to be concurrent, ensuring that even if multiple threads race to call getUUID for the first time, the expensive hardware query logic will only execute once. Subsequent calls from any thread will safely read the cached value.

## API Surface
The public contract is minimal, exposing only the functionality required to retrieve the identifier.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUUID() | UUID | O(N) first call, O(1) subsequent | Retrieves the machine's hardware UUID. The first invocation is an expensive operation involving I/O and process execution. All subsequent calls are constant time. Returns null if all retrieval methods fail. |

**WARNING:** The initial call to getUUID can block the calling thread for up to several seconds while it executes external processes. Avoid calling this method for the first time on a critical, time-sensitive thread such as the main render loop.

## Integration Patterns

### Standard Usage
The class is designed for straightforward, direct static invocation. The calling code **must** perform a null check on the result.

```java
// Retrieve the machine identifier for an analytics service
UUID machineId = HardwareUtil.getUUID();

if (machineId != null) {
    AnalyticsService.setMachineIdentifier(machineId.toString());
} else {
    // Handle the case where an ID could not be determined
    LOGGER.warn("Could not determine a stable machine identifier.");
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming a Non-Null Result:** Never assume getUUID will succeed. In sandboxed environments, restricted user accounts, or unusual system configurations, all retrieval methods can fail. Always check for null.
- **Misusing as a User Identifier:** This UUID identifies a *machine*, not a *user*. Using it as a primary key for user accounts is incorrect, as multiple users may share a single machine. It should only be used as a machine fingerprint.
- **Re-implementing Fallback Logic:** Do not attempt to replicate the OS-detection and fallback logic in client code. The purpose of this utility is to abstract away that complexity.

## Data Pipeline
HardwareUtil acts as a data source, not a processing stage. It originates a piece of data by interacting directly with the operating system.

> Flow:
> OS System Call (e.g., `reg query`, `ioreg`) or Filesystem Read (`/etc/machine-id`) -> Raw String Output -> **HardwareUtil (Parsing & Caching)** -> UUID Object -> Calling System (e.g., AnalyticsManager)

