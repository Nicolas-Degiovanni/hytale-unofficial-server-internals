---
description: Architectural reference for AssetEditorPacketHandler
---

# AssetEditorPacketHandler

**Package:** com.hypixel.hytale.builtin.asseteditor
**Type:** Session-Scoped Handler

## Definition
```java
// Signature
public class AssetEditorPacketHandler extends GenericPacketHandler {
```

## Architecture & Concepts
The AssetEditorPacketHandler is the primary server-side entry point for all network communication originating from a connected Hytale Asset Editor client. It serves as the translation and dispatch layer between the low-level network protocol and the high-level server logic.

For each active connection from an Asset Editor, a dedicated instance of this class is created. It is fundamentally a session-scoped object, encapsulating the state of a single client's connection, including their identity, permissions, and the underlying Netty channel for communication.

Its core responsibilities are:
1.  **Packet Demultiplexing:** It registers dozens of specific handler functions, each responsible for a single packet type (e.g., AssetEditorUpdateAsset, AssetEditorCreateDirectory). When a packet arrives, the base class GenericPacketHandler routes it to the correct typed method.
2.  **State Management:** It creates and owns the corresponding EditorClient object, which represents the authenticated user for the duration of the session.
3.  **Security Enforcement:** It performs mandatory permission checks for nearly every operation, ensuring that the connected user is authorized to perform the requested action. It is the first line of defense against unauthorized asset modifications.
4.  **Logic Delegation:** It does not contain business logic itself. Instead, it acts as a controller, delegating the actual work to the centralized AssetEditorPlugin or dispatching events onto the global server EventBus. This separation of concerns keeps the network layer clean and focused solely on communication protocols.

This class is a classic implementation of a stateful handler within a Netty-based server architecture. Its design ensures that client sessions are isolated and that network I/O operations do not block critical server threads.

### Lifecycle & Ownership
The lifecycle of an AssetEditorPacketHandler is tightly coupled to the underlying TCP connection from the client.

-   **Creation:** An instance is created by the server's core networking framework when a new client successfully establishes a connection and identifies itself as an Asset Editor. It is not instantiated directly by application logic.
-   **Scope:** The instance persists for the entire duration of the client's session. It lives as long as the Netty Channel is active and open.
-   **Destruction:** The handler is marked for garbage collection when the connection is terminated. The `closed` method is a critical lifecycle callback invoked by the Netty framework upon channel closure (due to client disconnect, server kick, or network fault). This callback ensures a clean shutdown by notifying the AssetEditorPlugin to release any resources associated with the disconnected EditorClient.

## Internal State & Concurrency
-   **State:** This class is stateful. Its primary state is the final reference to the EditorClient object, which encapsulates the user's identity and session data. It also holds a reference to the Netty Channel for outbound communication and inherits other connection-specific state from GenericPacketHandler.

-   **Thread Safety:** **WARNING:** This class is not thread-safe in a general sense, but it operates under the strict concurrency model of the Netty framework. Netty guarantees that all methods on a single handler instance are invoked by the *same* I/O thread (EventLoop). This confinement eliminates the need for explicit locking on its internal fields (like editorClient).

    However, it frequently dispatches work to other server systems, such as the EventBus, which execute on different threads. As seen in `handle(AssetEditorRequestDataset)`, it uses CompletableFuture to process logic asynchronously and then writes the response back to the channel. This is the correct pattern to avoid blocking the critical I/O thread. Any developer interacting with events originating from this handler must assume they are operating in a multi-threaded environment.

## API Surface
The public API is minimal, as the class is framework-controlled. Its primary interaction surface is the set of internal packet handling methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| handle(AssetEditorUpdateAsset) | void | O(N) | Delegates an asset content update to the AssetEditorPlugin. Complexity depends on asset size. |
| handle(AssetEditorRequestDataset) | void | O(1) | Asynchronously dispatches an event to the server EventBus to fetch data for the editor UI. |
| handle(AssetEditorActivateButton) | void | O(1) | Dispatches a generic button activation event to the EventBus for custom plugin handling. |
| handle(Disconnect) | void | O(1) | Handles a client-initiated disconnect notification, logging the reason and closing the channel. |
| getEditorClient() | EditorClient | O(1) | Returns the stateful object representing the connected client. |

## Integration Patterns

### Standard Usage
Developers typically do not interact with AssetEditorPacketHandler directly. Instead, they interact with the *results* of its operations by listening for events on the server's EventBus. The handler acts as the bridge between a network packet and a server-side event.

```java
// A plugin developer would listen for an event, not call the handler.
// This event is fired from within handle(AssetEditorActivateButton).

@Subscribe
public void onEditorButtonClick(AssetEditorActivateButtonEvent event) {
    // The 'event' object contains the EditorClient, which originated from the handler.
    EditorClient client = event.getEditorClient();
    String buttonId = event.getButtonId();

    if (buttonId.equals("myplugin.mybutton")) {
        client.sendPopupNotification(AssetEditorPopupNotificationType.Info, "My button was clicked!");
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new AssetEditorPacketHandler()`. This object is deeply tied to the Netty channel lifecycle and must be instantiated by the server's connection management framework. Manual creation will result in a non-functional handler.
-   **Blocking Operations:** Never introduce blocking code (e.g., database queries, heavy file I/O) directly inside a `handle` method. Doing so will stall the Netty I/O thread, severely degrading network performance for all connected clients on that thread. Use the server's asynchronous systems like the EventBus or CompletableFuture.
-   **Bypassing Permission Checks:** The `lacksPermission` checks are a critical security boundary. Removing or circumventing these checks would allow any connected user to modify any server asset, leading to severe security vulnerabilities.

## Data Pipeline
The AssetEditorPacketHandler is a central node in the data flow between the Asset Editor client and the server backend.

**Example 1: Asset Modification Request**
> Flow:
> Client Action -> AssetEditorUpdateAsset Packet -> Netty I/O Thread -> **AssetEditorPacketHandler**.handle(AssetEditorUpdateAsset) -> Permission Check -> AssetEditorPlugin.handleAssetUpdate -> Filesystem Write -> (Reply Packet) -> **AssetEditorPacketHandler**.getEditorClient().write(...) -> Netty Channel -> Client UI

**Example 2: Asynchronous Data Request**
> Flow:
> Client UI Request -> AssetEditorRequestDataset Packet -> Netty I/O Thread -> **AssetEditorPacketHandler**.handle(AssetEditorRequestDataset) -> Server EventBus Dispatch (Async) -> Other Module/Plugin (Worker Thread) -> Data Retrieval -> CompletableFuture.thenAccept(...) -> (Reply Packet) -> **AssetEditorPacketHandler**.getEditorClient().write(...) -> Netty Channel -> Client UI

