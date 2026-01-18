---
description: Architectural reference for AssetUtil
---

# AssetUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public class AssetUtil {
```

## Architecture & Concepts
AssetUtil is a static utility class providing a simplified, high-level entry point for accessing server asset file paths. Architecturally, it serves as a static facade over the more complex and stateful AssetModule service.

Its primary purpose is to offer a zero-configuration, globally accessible method for retrieving the root directory of the server's base asset pack. However, this design pattern couples code directly to a static utility, hindering testability and obscuring the true dependency on the asset system.

**WARNING:** This class is deprecated and marked for removal. Its existence is a temporary bridge for legacy code. All new development must interface directly with the AssetModule service, which should be retrieved from the server's service context.

## Lifecycle & Ownership
As a class containing only static members, AssetUtil has no instance lifecycle.

- **Creation:** The class is loaded into the JVM by the ClassLoader upon its first reference. No instances are ever created.
- **Scope:** Static. The class definition persists for the entire lifetime of the server application once loaded.
- **Destruction:** The class is unloaded from the JVM during application shutdown.

## Internal State & Concurrency
- **State:** AssetUtil is entirely stateless. It maintains no internal fields and does not cache any data. Each invocation of its methods delegates immediately to the underlying AssetModule.
- **Thread Safety:** This class is inherently thread-safe. The `getHytaleAssetsPath` method is a pure function from the perspective of this class, containing no mutable state or side effects. Thread safety is therefore dependent on the thread safety of the downstream AssetModule, which is designed for concurrent access.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHytaleAssetsPath() | Path | O(1) | **DEPRECATED.** Retrieves the root file system path for the base asset pack by delegating to the AssetModule singleton. |

## Integration Patterns

### Standard Usage
The standard pattern is to **avoid this class entirely** and instead retrieve the AssetModule from the service context. This ensures proper dependency management and testability.

```java
// CORRECT: Retrieve the AssetModule service from the context
AssetModule assetModule = serverContext.getService(AssetModule.class);

// Use the service directly to get asset information
Path assetRoot = assetModule.getBaseAssetPack().getRoot();
```

### Anti-Patterns (Do NOT do this)
- **Usage in New Code:** Do not call `AssetUtil.getHytaleAssetsPath()` in any new code. This introduces a dependency on a component that will be removed, creating future build failures and technical debt.
- **Ignoring Deprecation:** Continued reliance on this utility class is the primary anti-pattern. All existing usages should be scheduled for refactoring to use the standard dependency injection pattern shown above.

## Data Pipeline
AssetUtil is not a data processing component; it is a simple accessor. The flow represents a request for data rather than a transformation pipeline.

> Flow:
> Caller -> **AssetUtil.getHytaleAssetsPath()** -> AssetModule (Singleton Access) -> BaseAssetPack Instance -> Root Path Property

