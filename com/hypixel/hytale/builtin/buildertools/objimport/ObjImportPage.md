---
description: Architectural reference for ObjImportPage
---

# ObjImportPage

**Package:** com.hypixel.hytale.builtin.buildertools.objimport
**Type:** Transient

## Definition
```java
// Signature
public class ObjImportPage extends InteractiveCustomUIPage<ObjImportPage.PageData> {
```

## Architecture & Concepts
The ObjImportPage class is a server-side controller that manages the user interface for importing 3D models in Wavefront OBJ format. It functions as a stateful bridge between the client-side UI, defined in *Pages/ObjImportPage.ui*, and the server's backend processing systems.

Architecturally, its primary responsibility is to gather user-defined parameters, validate them, and orchestrate a complex, resource-intensive import process without blocking the main server thread. This is achieved by delegating the core import logic to the BuilderToolsPlugin's asynchronous task queue.

The class encapsulates the entire import workflow:
1.  **UI State Management:** It maintains the state of all UI elements, such as file paths, scaling options, and block patterns.
2.  **File Browsing:** It integrates a ServerFileBrowser component to provide a secure, server-side file selection interface, restricted to the designated *imports/models* directory.
3.  **Event Handling:** It processes all user interactions from the client via the handleDataEvent method, updating its internal state accordingly.
4.  **Asynchronous Processing:** Upon user confirmation, it captures the current configuration and submits a task to a dedicated worker queue. This task handles file parsing (ObjParser, MtlParser), mesh-to-block conversion (MeshVoxelizer), and material-to-block mapping (BlockColorIndex).
5.  **Result Integration:** The result of the asynchronous task, a BlockSelection object, is placed into the player's builder state (clipboard). The class then attempts to automatically switch the player's active tool to the Paste Tool for immediate use.

This component is a critical part of the creative toolset, enabling players and creators to bring external 3D assets into the game world as block-based structures.

### Lifecycle & Ownership
-   **Creation:** An instance of ObjImportPage is created for a specific player when they are directed to this UI. This is typically initiated by a server command or another UI action within the BuilderToolsPlugin. The instance is tied directly to a PlayerRef.
-   **Scope:** The object's lifetime is bound to the player's interaction with the OBJ Import UI. It persists as long as the page is active for that player.
-   **Destruction:** The instance is marked for garbage collection when the player closes the UI, or upon successful import when the PageManager transitions the player to a different page (e.g., Page.None).

## Internal State & Concurrency
-   **State:** The class is highly stateful and mutable. It holds numerous fields representing the current values of the UI form inputs (e.g., objPath, targetHeight, useScaleMode). It also tracks the operational state of the UI, such as isProcessing and statusMessage. This state is modified exclusively in response to events from the associated player.

-   **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread responsible for the player's updates. The core design explicitly avoids concurrency issues by offloading the heavy computational work. The performImport method captures a final, immutable snapshot of the UI parameters and passes them to a background task. This prevents race conditions where a player could modify UI settings while an import is already in progress.

    **WARNING:** Direct modification of this object's state from other threads will lead to server instability and data corruption.

## API Surface
The public contract is defined by its parent class, InteractiveCustomUIPage. Direct invocation of its methods is not a standard use case; the server's UI framework manages its lifecycle and event flow.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(ref, commandBuilder, eventBuilder, store) | void | O(N) | Called by the UI framework to generate rendering and event binding commands for the client. Complexity is proportional to the number of UI elements. |
| handleDataEvent(ref, store, data) | void | O(1) | The primary entry point for client-side events. Decodes PageData and updates internal state or triggers the import process. |

## Integration Patterns

### Standard Usage
This class is not meant to be instantiated or managed directly. It is set as a player's active page through the Player's PageManager.

```java
// Correctly display the ObjImportPage for a player
PlayerRef playerRef = ...;
Player player = ...;

// The framework will create and manage the page instance
player.getPageManager().setPage(ref, store, new ObjImportPage(playerRef));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation and Management:** Do not create an instance and attempt to call its methods manually. The class relies on the context and lifecycle provided by the PageManager.
-   **Asynchronous State Modification:** Do not attempt to modify the page's state from another thread, especially while an import is processing. The state is not designed for concurrent access.
-   **Bypassing the Task Queue:** Do not extract the import logic to run it on the main server thread. This will cause severe server lag and unresponsiveness.

## Data Pipeline
The flow of data from user action to in-game result is a multi-stage, asynchronous process.

> Flow:
> Client UI Event -> Network Packet -> Server UI Framework -> **ObjImportPage.handleDataEvent** -> **ObjImportPage.performImport** -> BuilderToolsPlugin Task Queue -> Worker Thread (Parses OBJ/MTL, Voxelizes Mesh) -> BlockSelection Object -> Player Builder State (Clipboard) -> Network Packet -> Client Tool Update

