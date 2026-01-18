---
description: Architectural reference for EditorClient
---

# EditorClient

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Transient

## Definition
```java
// Signature
public class EditorClient implements PermissionHolder {
```

## Architecture & Concepts
The EditorClient class is a server-side representation of a remote user connected to the Hytale asset editor. It serves as a lightweight, session-scoped proxy that encapsulates the identity, permissions, and communication channel for a single editor instance.

This class is a critical abstraction that decouples the asset editing subsystems from the core game server's player management logic (e.g., the Universe and PlayerRef). While an EditorClient may be associated with an in-game player, it is not a requirement; the editor can be used in a standalone context.

Its primary responsibilities are:
1.  **Identity Management:** Holding the UUID and username of the connected user.
2.  **Security Principal:** Implementing the PermissionHolder interface to serve as the subject for all permission checks related to asset editing operations. It delegates these checks to the global PermissionsModule.
3.  **Communication Endpoint:** Providing a high-level API for sending structured replies and notifications back to the remote client via its encapsulated PacketHandler.

## Lifecycle & Ownership
-   **Creation:** An EditorClient instance is created by the server's session management layer when a user successfully establishes a connection to the asset editor service. It is not intended for direct instantiation by business logic. The multiple constructors support different authentication contexts, such as a fully authenticated player or a more abstract system user.
-   **Scope:** The object's lifetime is strictly bound to the duration of the asset editor's network session. It persists as long as the client is connected.
-   **Destruction:** The object is eligible for garbage collection once the corresponding network session is terminated and all references to it are released. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** EditorClient is a stateful, mutable object. It holds immutable identifiers like UUID and username, but also contains mutable state such as the client's language. It does not perform any internal caching; it acts as a container for session state and service handles.
-   **Thread Safety:** This class is **not thread-safe**. It contains no internal synchronization mechanisms. It is designed to be accessed exclusively by the single thread responsible for processing requests for its corresponding network session.

    **WARNING:** Modifying or accessing an EditorClient instance from multiple threads concurrently will lead to race conditions and unpredictable behavior, especially when calling setLanguage or interacting with the underlying PacketHandler. All interactions must be synchronized externally or, preferably, confined to a single-threaded execution model per session.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryGetPlayer() | PlayerRef | O(log N) | Attempts to resolve the client to an active in-game player from the Universe. Returns null if the player is not currently in the world. |
| hasPermission(id) | boolean | O(log N) | Checks if the client has a specific permission. Delegates to the global PermissionsModule. |
| sendPopupNotification(type, message) | void | O(1) | Sends a non-blocking UI notification to the remote editor. |
| sendSuccessReply(token, message) | void | O(1) | Sends a success reply packet, acknowledging a specific request identified by the token. |
| sendFailureReply(token, message) | void | O(1) | Sends a failure reply packet, rejecting a specific request identified by the token. |

## Integration Patterns

### Standard Usage
The EditorClient is typically provided as a context object to handlers that process incoming requests from the asset editor. The handler uses the client to verify permissions and then sends a response.

```java
// Example of a command handler receiving an EditorClient
public void handleAssetSaveRequest(EditorClient client, int requestToken, AssetData data) {
    if (!client.hasPermission("asset.editor.save")) {
        client.sendFailureReply(requestToken, new Message("permission.denied"));
        return;
    }

    // ... process and save the asset ...

    client.sendSuccessReply(requestToken, new Message("asset.saved.success"));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new EditorClient()`. The session management layer is the sole owner of its lifecycle. Always use the instance provided by the request handling framework.
-   **State Sharing Across Sessions:** Never share an EditorClient instance between different user sessions or threads. Each instance is tied to a unique connection.
-   **Assuming Player Presence:** Do not call `tryGetPlayer()` and expect a non-null result. The editor can be used without the user being logged into a game world. Always perform a null check.
-   **Using Deprecated Constructors:** Avoid using the `EditorClient(PlayerRef playerRef)` constructor. It creates a tight coupling to the game universe that this class is designed to avoid.

## Data Pipeline
The EditorClient acts as a final step in the server-side logic, translating high-level actions into low-level network packets for the remote client.

> Flow:
> Asset Editor Command Handler -> **EditorClient**.sendReply() -> PacketHandler.write(Packet) -> Server Network Layer -> TCP/IP Stack -> Remote Editor Client UI

