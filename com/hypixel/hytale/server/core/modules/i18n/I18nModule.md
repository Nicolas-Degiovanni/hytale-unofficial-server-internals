---
description: Architectural reference for I18nModule
---

# I18nModule

**Package:** com.hypixel.hytale.server.core.modules.i18n
**Type:** Singleton

## Definition
```java
// Signature
public class I18nModule extends JavaPlugin {
```

## Architecture & Concepts
The I18nModule is the centralized service for managing all server-side internationalization (i18n) and localization (l10n). As a core `JavaPlugin`, it integrates deeply with the server's asset and event systems to provide a dynamic, reloadable translation layer.

Its primary responsibilities are:
1.  **Asset-Driven Loading:** It discovers and parses `.lang` files from all registered `AssetPack` directories, structuring them into a hierarchical, in-memory database of translation keys.
2.  **Dynamic Key Generation:** It listens for `LoadedAssetsEvent` broadcasts for specific asset types (e.g., `BlockType`, `ResourceType`). It then dynamically generates and injects new translation keys based on the properties of these assets, such as crafting category names or resource descriptions. This decouples game content from static language files.
3.  **Client Synchronization:** It is responsible for serializing and transmitting the complete translation table to clients upon connection (`UpdateTranslations` with `UpdateType.Init`).
4.  **Live Reloading:** It integrates with the `AssetMonitor` to detect real-time changes to `.lang` files on disk. Upon modification, it calculates the delta (added, updated, and removed keys) and broadcasts granular `UpdateTranslations` packets to all connected clients, enabling localization changes without a server restart.
5.  **Fallback Logic:** It manages a fallback system, ensuring that if a key is missing in a specific language (e.g., `es-MX`), it will attempt to resolve it from a more general language (`en-US` by default) as defined in `fallback.lang`.

This module acts as the single source of truth for all display text on the server, ensuring consistency and enabling dynamic content localization.

### Lifecycle & Ownership
-   **Creation:** A single instance is created by the server's plugin loader during the bootstrap sequence. The static `instance` field is set within the constructor, enforcing the Singleton pattern.
-   **Scope:** The I18nModule instance is application-scoped and persists for the entire lifetime of the server process.
-   **Destruction:** The module is destroyed and garbage collected during server shutdown when the plugin system is dismantled. There is no explicit public cleanup method.

## Internal State & Concurrency
-   **State:** The internal state is highly mutable and is composed of three primary data structures:
    -   `languages`: A `ConcurrentHashMap` storing the raw key-value pairs for each loaded language, keyed by language code (e.g., "en-US").
    -   `fallbacks`: A `ConcurrentHashMap` mapping language codes to their fallback language code.
    -   `cachedLanguages`: A `ConcurrentHashMap` used as a performance cache. It stores fully resolved, unmodifiable maps for each language, with all fallback keys already merged. This cache is invalidated when underlying language files change.

-   **Thread Safety:** The class is designed to be thread-safe. The core state-holding maps are concurrent collections (`ConcurrentHashMap`). This is critical as asset loading, live file monitoring, and player connection handling may occur on different threads, all of which need to read from or modify the translation state. Access to the maps is managed internally to prevent data corruption.

## API Surface
The public API is minimal, exposing only high-level operations for retrieving translations and initiating synchronization. Direct manipulation of the internal language maps is not permitted.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | I18nModule | O(1) | Static accessor for the singleton instance. |
| getMessages(language) | Map<String, String> | O(N) | Returns an unmodifiable, cached map of all keys for a given language, including fallbacks. The first call for a language incurs a computation cost to build the cache. |
| sendTranslations(handler, language) | void | O(M) | Serializes the complete translation map for a language and sends it to a client via the provided PacketHandler. M is the total number of translation keys. |
| getMessage(language, key) | String | O(1) | Retrieves a single translated string for a given language and key. Returns null if not found. This is the primary lookup method. |

## Integration Patterns

### Standard Usage
The module should always be accessed via its static `get()` method. The most common use case is retrieving a specific translation key for a player's configured language.

```java
// Retrieve the module instance
I18nModule i18n = I18nModule.get();

// Get a message for a player's language
// The player's language code should be retrieved from their session data
String playerLanguage = "en-US";
String welcomeMessage = i18n.getMessage(playerLanguage, "server.login.welcome");

if (welcomeMessage != null) {
    // Send message to player
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new I18nModule()`. The server's plugin framework is solely responsible for its lifecycle. Attempting to create a second instance will break the singleton pattern and lead to unpredictable behavior.
-   **Cache Reliance During Reloads:** Do not fetch a language map, store it long-term, and assume it is still valid. The live reload mechanism can invalidate the data at any time. Always request messages from the module directly to ensure you receive the most up-to-date translations.
-   **Assuming Key Existence:** Do not assume a translation key exists, especially for dynamically generated keys from assets. Always perform a null check on the result of `getMessage`.

## Data Pipeline
The data flow for the live reload feature demonstrates the module's reactive nature. It shows how a change on the filesystem propagates through the server and out to connected clients.

> Flow:
> Filesystem Event (file modified) -> AssetMonitor -> **I18nAssetMonitorHandler** -> Calculate Delta (removed/changed keys) -> **I18nModule.getUpdatePacketsForChanges()** -> UpdateTranslations Packet -> PlayerRef PacketHandler -> Network -> Client UI Update

---

