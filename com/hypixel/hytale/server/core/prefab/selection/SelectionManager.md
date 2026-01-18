---
description: Architectural reference for SelectionManager
---

# SelectionManager

**Package:** com.hypixel.hytale.server.core.prefab.selection
**Type:** Utility

## Definition
```java
// Signature
public final class SelectionManager {
```

## Architecture & Concepts
The SelectionManager is a static utility class that acts as a global service locator for the active SelectionProvider implementation. Its primary architectural purpose is to decouple game systems that *consume* selection logic from the system that *provides* it.

By acting as a central, static point of access, any part of the server codebase can retrieve the current selection handling strategy without requiring direct dependencies on a concrete implementation. This follows the Inversion of Control principle, where a high-level component (the SelectionManager) is not responsible for creating its dependencies but instead receives them from an external source at runtime.

This component is critical for features that rely on spatial or entity selection, such as administrative tools, building mechanics, or scripted events. It ensures that all parts of the server use a single, authoritative source for selection operations.

### Lifecycle & Ownership
- **Creation:** The SelectionManager class is never instantiated due to its private constructor. Its static state is initialized by the Java Virtual Machine during class loading. The contained SelectionProvider is created and injected externally, typically by a core server system or a high-priority plugin during the server's bootstrap sequence.
- **Scope:** The static reference to the SelectionProvider persists for the entire lifetime of the server process.
- **Destruction:** The reference is only cleared upon JVM shutdown when the class is unloaded. There is no explicit cleanup method.

## Internal State & Concurrency
- **State:** The class maintains a single, mutable static field: a reference to the currently active SelectionProvider. This state is held within an AtomicReference, indicating that it is expected to be accessed and potentially modified in a concurrent environment.
- **Thread Safety:** This class is thread-safe. The use of AtomicReference guarantees that the read and write operations on the underlying SelectionProvider reference are atomic. This prevents race conditions where multiple threads might attempt to get or set the provider simultaneously, ensuring memory visibility and atomicity without explicit locking.

**WARNING:** While the manager itself is thread-safe, the thread safety of the *injected SelectionProvider implementation* is not guaranteed by this class and is the responsibility of the provider's author.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setSelectionProvider(provider) | void | O(1) | Sets the global instance of the SelectionProvider. This will overwrite any previously set provider. |
| getSelectionProvider() | SelectionProvider | O(1) | Retrieves the currently configured SelectionProvider. Returns null if no provider has been set. |

## Integration Patterns

### Standard Usage
A core system or plugin must register its implementation of SelectionProvider during its initialization phase. Other systems can then retrieve and use this provider as needed.

```java
// In a plugin's initialization logic
CustomSelectionProvider myProvider = new CustomSelectionProvider();
SelectionManager.setSelectionProvider(myProvider);

// In a game logic system or command handler
SelectionProvider provider = SelectionManager.getSelectionProvider();
if (provider != null) {
    // Perform selection-related operations
    provider.createSelectionForPlayer(player);
} else {
    // Handle the case where no selection system is available
    log.warn("Selection system is not available.");
}
```

### Anti-Patterns (Do NOT do this)
- **Assuming Non-Nullability:** Do not call getSelectionProvider and use the result without a null check. The provider may not have been initialized, leading to a NullPointerException.
- **Hot-Swapping Providers:** Do not repeatedly call setSelectionProvider during the main game loop. While thread-safe, the design implies a single provider is registered at startup. Frequent changes can lead to unpredictable behavior for consumers who may have cached an old provider instance.

## Data Pipeline
The SelectionManager does not process data. It acts as a static container and access point for a service implementation.

> **Registration Flow:**
> Core Server / Plugin -> Creates `SelectionProvider` instance -> `setSelectionProvider()` -> **SelectionManager** (stores reference in AtomicReference)

> **Retrieval Flow:**
> Game System / Command -> `getSelectionProvider()` -> **SelectionManager** (returns reference) -> Caller uses `SelectionProvider` instance

