---
description: Architectural reference for FileBrowserConfig
---

# FileBrowserConfig

**Package:** com.hypixel.hytale.server.core.ui.browser
**Type:** Immutable Configuration Object

## Definition
```java
// Signature
public record FileBrowserConfig(
   @Nonnull String listElementId,
   @Nullable String rootSelectorId,
   @Nullable String searchInputId,
   @Nullable String currentPathId,
   @Nonnull List<FileBrowserConfig.RootEntry> roots,
   @Nonnull Set<String> allowedExtensions,
   boolean enableRootSelector,
   boolean enableSearch,
   boolean enableDirectoryNav,
   boolean enableMultiSelect,
   int maxResults,
   @Nullable FileListProvider customProvider
)
```

## Architecture & Concepts

FileBrowserConfig is a foundational data structure, not an active service. It functions as a parameter object that declaratively defines the behavior, appearance, and data sources for a server-driven file browser user interface. Its primary role is to decouple the logic that *requests* a file browser (e.g., a world selection screen) from the UI system that *renders* and manages the browser component.

This class is designed as an immutable Java Record, which guarantees that a configuration, once created, is a stable and predictable contract. Construction is managed exclusively through the nested Builder class, enforcing valid state and providing a fluent, readable API for assembly.

Architecturally, it employs two key design patterns:
1.  **Builder Pattern:** The `FileBrowserConfig.Builder` provides a robust and self-documenting mechanism for creating complex configuration objects. This prevents constructor over-provisioning and ensures required fields are set.
2.  **Strategy Pattern:** The optional `FileListProvider` field allows developers to inject custom logic for sourcing file and directory listings. If null, a default file-system provider is used. If present, it completely overrides the standard behavior, enabling integration with virtual file systems, remote storage, or database-backed asset lists.

## Lifecycle & Ownership

-   **Creation:** An instance is created on-demand by a higher-level system that needs to display a file browser to the user. Instantiation is always performed via the `FileBrowserConfig.builder()` factory method and the subsequent `build()` call.
-   **Scope:** Short-lived and transient. A FileBrowserConfig object typically exists only for the duration of a single UI interaction. It is constructed, passed to the UI rendering engine, and then becomes eligible for garbage collection.
-   **Destruction:** There is no explicit destruction method. As a simple data object with no managed resources, its memory is reclaimed by the Java Garbage Collector when it is no longer referenced.

## Internal State & Concurrency

-   **State:** **Immutable**. As a Java Record, all fields are final. Once an instance is created by the builder, its state cannot be modified. This design choice eliminates entire categories of bugs related to state corruption and race conditions.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutability, a FileBrowserConfig instance can be safely shared and read by multiple threads without any external synchronization or locking mechanisms. It is a safe object to pass between service threads and the main UI thread.

## API Surface

The primary public contract is the nested Builder class. The record itself only provides implicit accessor methods for its fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder() | Builder | O(1) | Static factory method to create a new configuration builder. |
| build() | FileBrowserConfig | O(1) | Terminal operation on the builder. Validates and constructs the immutable config object. |
| roots(roots) | Builder | O(1) | Sets the list of starting directories the user can browse. |
| allowedExtensions(ext) | Builder | O(N) | Defines the set of file extensions to display. Filters out all other files. |
| customProvider(provider) | Builder | O(1) | **CRITICAL:** Injects a custom file listing strategy, overriding default filesystem access. |
| enableMultiSelect(bool) | Builder | O(1) | Configures the UI to allow single or multiple file selection. |

## Integration Patterns

### Standard Usage

The intended use is to construct a configuration fluently when a file browser is needed, and pass the resulting object to the UI system.

```java
// Example: Configure a browser to select a Hytale world save file.
FileBrowserConfig worldSelectorConfig = FileBrowserConfig.builder()
    .listElementId("#WorldListPanel")
    .rootSelectorId("#WorldLocationDropdown")
    .searchInputId("#WorldSearch")
    .roots(List.of(
        new FileBrowserConfig.RootEntry("My Worlds", Paths.get("./worlds")),
        new FileBrowserConfig.RootEntry("Downloaded Worlds", Paths.get("./downloads/worlds"))
    ))
    .allowedExtensions("hytale")
    .enableMultiSelect(false)
    .maxResults(100)
    .build();

// This config object is then passed to the UI service.
uiService.showFileBrowser(worldSelectorConfig);
```

### Anti-Patterns (Do NOT do this)

-   **State Re-use and Modification:** Do not cache and attempt to reuse a FileBrowserConfig instance for different contexts. Because it is immutable, you cannot change it. Instead, create a new configuration with the desired parameters for each use case.
-   **Invalid UI Identifiers:** Providing non-existent or incorrect element IDs for fields like `listElementId` will not cause a server-side crash but will result in a non-functional UI component on the client. The UI will fail to bind its interactive elements.
-   **Blocking FileListProvider:** If implementing a custom `FileListProvider`, ensure its methods are non-blocking and performant. A slow provider will freeze the UI thread, leading to a poor user experience. Offload heavy I/O operations to a background thread.

## Data Pipeline

FileBrowserConfig does not process a stream of data. Instead, it *is* the data that initiates and configures a UI process.

> Flow:
> Server-Side Service (e.g., WorldManager) -> **FileBrowserConfig.Builder** -> **FileBrowserConfig Instance** -> UI Service -> UI Renderer -> Client-Side Component (Binds to element IDs)

