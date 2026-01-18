---
description: Architectural reference for AssetNotifications
---

# AssetNotifications

**Package:** com.hypixel.hytale.server.core.asset
**Type:** Utility

## Definition
```java
// Signature
public class AssetNotifications {
```

## Architecture & Concepts
The AssetNotifications class is a static contract, not an active system component. It serves as a centralized, immutable schema for defining the presentation details of asset-related user notifications. Its primary architectural role is to decouple the systems that *trigger* asset events (like AssetStore or live-reloading file watchers) from the systems that *render* notifications to the user (like the UI layer).

By providing a single source of truth for message keys, icon paths, and color codes, it ensures a consistent look and feel for all asset-related feedback across the application. Systems that need to create a notification do not need to know the specific icon file or hex color; they only need to reference the appropriate constant from this class. This pattern prevents the proliferation of hard-coded magic strings and simplifies future updates to notification styling.

## Lifecycle & Ownership
As a class containing only static final constants, AssetNotifications does not follow a traditional object lifecycle.

- **Creation:** The class is not instantiated. It is loaded into the JVM by the ClassLoader on its first access by any other component in the system. This is a one-time event for the application's lifetime.
- **Scope:** The class and its constants exist in a global, application-wide scope for the entire duration of the server process.
- **Destruction:** The class is unloaded from memory only when the application's ClassLoader is garbage collected, which typically occurs during a full server shutdown.

## Internal State & Concurrency
- **State:** This class is stateless. All its members are `public static final String` literals, which are compile-time constants. Their values are immutable and cannot be changed at runtime.
- **Thread Safety:** The class is inherently and unconditionally thread-safe. As it contains only immutable static constants, it can be safely accessed and read from any number of concurrent threads without requiring locks or any other synchronization mechanisms.

## API Surface
The public contract consists entirely of static constants used to construct UI notifications.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ASSET_ADDED_MESSAGE_KEY | String | O(1) | Localization key for the "asset added" message. |
| ASSET_DELETED_MESSAGE_KEY | String | O(1) | Localization key for the "asset deleted" message. |
| ASSET_RELOADED_MESSAGE_KEY | String | O(1) | Localization key for the "asset reloaded" message. |
| ASSET_SECONDARY_GENERIC_MESSAGE_KEY | String | O(1) | Secondary localization key for generic asset messages. |
| ASSET_ADDED_ICON | String | O(1) | Resource path for the "asset added" notification icon. |
| ASSET_DELETED_ICON | String | O(1) | Resource path for the "asset deleted" notification icon. |
| ASSET_RELOADED_ICON | String | O(1) | Resource path for the "asset reloaded" notification icon. |
| ASSET_ADDED_COLOR | String | O(1) | Hex color code for "asset added" notifications. |
| ASSET_DELETED_COLOR | String | O(1) | Hex color code for "asset deleted" notifications. |
| ASSET_RELOADED_COLOR | String | O(1) | Hex color code for "asset reloaded" notifications. |

## Integration Patterns

### Standard Usage
This class should be used by any system responsible for creating user-facing notifications in response to asset lifecycle events. The consumer references the constants to populate a notification data structure.

```java
// Example: A hypothetical NotificationService building a notification
// after an asset was successfully reloaded.

public void onAssetReloaded(String assetName) {
    Notification.Builder builder = new Notification.Builder();

    // Use the constants to define the notification's appearance
    builder.setMessageKey(AssetNotifications.ASSET_RELOADED_MESSAGE_KEY);
    builder.setIcon(AssetNotifications.ASSET_RELOADED_ICON);
    builder.setColor(AssetNotifications.ASSET_RELOADED_COLOR);
    builder.addMessageArgument(assetName); // e.g., "sword.json"

    notificationManager.dispatch(builder.build());
}
```

### Anti-Patterns (Do NOT do this)
- **Hard-Coding Literals:** Never use the raw string values elsewhere in the codebase. This defeats the purpose of the class and creates a maintenance nightmare.
  ```java
  // BAD: Creates technical debt
  builder.setMessageKey("server.general.assetstore.reloadAssets");
  builder.setColor("#A7AfA7");
  ```
- **Attempted Instantiation:** This class is not designed to be instantiated. It has no instance fields or methods.
  ```java
  // BAD: Will not compile and is conceptually incorrect
  AssetNotifications notifications = new AssetNotifications();
  ```

## Data Pipeline
This class does not process data; it *defines* the schema for data that flows from the asset system to the UI system.

> Flow:
> AssetStore Event (e.g., File Reloaded) -> Notification Service -> **AssetNotifications (Constants Referenced)** -> Notification Payload Created -> UI System -> Rendered Notification

