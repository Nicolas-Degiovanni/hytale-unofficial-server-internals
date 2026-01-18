---
description: Architectural reference for PrototypePlayerBuilderToolSettings
---

# PrototypePlayerBuilderToolSettings

**Package:** com.hypixel.hytale.builtin.buildertools
**Type:** Session-Scoped State Object

## Definition
```java
// Signature
public class PrototypePlayerBuilderToolSettings {
```

## Architecture & Concepts
The PrototypePlayerBuilderToolSettings class is a stateful data container that encapsulates all transient settings and data for a single player's builder tools. It is not a service or a manager; it is a Plain Old Java Object whose sole responsibility is to hold state.

This class acts as a central repository for a player's creative mode context, decoupling complex tool state from the core Player entity. This includes:
-   **Brush Configuration:** The active BrushConfig object and its associated command executor.
-   **Selection & Clipboard:** Caches block and fluid data for copy/paste operations via the selection tool.
-   **Operation History:** Maintains a list of recent paint operations to prevent re-painting the same block, effectively acting as a short-term "mask" or "undo" buffer.
-   **UI State:** Flags that control the visibility of editor settings for the player.

Each player in the game has a distinct and separate instance of this class, associated via their unique UUID. The lifecycle of this object is managed externally, typically by the ToolOperation system, which acts as a factory and registry.

## Lifecycle & Ownership
-   **Creation:** Lazily instantiated by the static factory method `ToolOperation.getOrCreatePrototypeSettings(UUID)`. An instance is created on-demand the first time a player interacts with a builder tool that requires persistent settings.
-   **Scope:** The object is session-scoped. It persists for as long as the player is connected to the server. Its state is tied to the player's UUID, ensuring settings are maintained across different interactions within the same game session.
-   **Destruction:** There is no explicit destruction or cleanup method. The object is eligible for garbage collection when the external manager (within the ToolOperation system) releases its reference, which typically occurs when a player disconnects from the server.

## Internal State & Concurrency
-   **State:** This object is highly **mutable**. Its primary purpose is to cache data and track changing user settings. It holds large, mutable data structures, such as arrays of BlockChange for clipboard data and a LinkedList of LongOpenHashSet for operation history.
-   **Thread Safety:** **This class is not thread-safe.** It contains no internal synchronization mechanisms (e.g., locks, volatile fields, or concurrent collections). All interactions with an instance of this class **must** be performed on the main server thread. Unsynchronized access from other threads will lead to race conditions, data corruption, and unpredictable server behavior.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setInSelectionTransformationMode(boolean) | void | O(1) | Toggles the player's selection paste mode. Critically, this method also cleans up clipboard data when exiting the mode. |
| setBlockChangesForPlaySelectionToolPasteMode(BlockChange[]) | void | O(1) | Caches an array of block changes, effectively acting as the "copy" part of a copy-paste operation. |
| addIgnoredPaintOperation() | LongOpenHashSet | O(1) | Creates and adds a new set to the operation history list for a new paint action. Manages history length. |
| containsLocation(int, int, int) | boolean | O(N) | Scans the entire operation history to check if a block position has been recently modified. N is the number of historical operations. |
| isOkayToDoCommandsOnSelection(Ref, Player, ComponentAccessor) | static boolean | O(1) | A static utility to validate if a player can perform a world modification command, preventing conflicts with an active paste operation. |

## Integration Patterns

### Standard Usage
The object should never be instantiated directly. It must be retrieved via the `ToolOperation` factory method using the target player's UUID. It is then used as a data source or a state sink for builder tool logic.

```java
// In a command executor or tool handler
UUID playerUUID = player.getUUID();

// Retrieve the session-specific settings object
PrototypePlayerBuilderToolSettings settings = ToolOperation.getOrCreatePrototypeSettings(playerUUID);

// Modify state based on player action
if (isCopyAction) {
    BlockChange[] selection = calculateSelection(player);
    settings.setBlockChangesForPlaySelectionToolPasteMode(selection);
}

// Read state to perform an operation
if (isPasteAction) {
    BlockChange[] clipboard = settings.getBlockChangesForPlaySelectionToolPasteMode();
    if (clipboard != null) {
        applyChangesToWorld(clipboard);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PrototypePlayerBuilderToolSettings(uuid)`. This bypasses the central registry, leading to multiple, conflicting settings objects for the same player and causing state desynchronization.
-   **Multi-threaded Access:** Do not read or write to this object from any thread other than the main server thread. This will cause critical race conditions.
-   **State Negligence:** Modifying one piece of state without considering its relationship to others. For example, setting `blockChangesForPlaySelectionToolPasteMode` to a non-null value without also setting `isInSelectionTransformationMode` to true can leave the system in an inconsistent state.

## Data Pipeline
This class is not a pipeline processor itself, but rather a stateful component that systems in a data pipeline interact with. It serves as a temporary data store between different stages of a player-driven world editing operation.

> **Copy/Paste Flow:**
> Player Input (`/copy` command) -> Command Handling System -> **PrototypePlayerBuilderToolSettings.setBlockChanges...** (State is stored)
>
> Player Input (`/paste` command) -> Command Handling System -> Read from **PrototypePlayerBuilderToolSettings** -> World Edit System -> Block Update Packets

> **Brush Painting Flow:**
> Player Input (Mouse Click) -> Network Packet -> Server Tick -> Tool Operation Handler -> Read `BrushConfig` from **PrototypePlayerBuilderToolSettings** -> Calculate Affected Blocks -> World Edit System -> Add to `ignoredPaintOperations` in **PrototypePlayerBuilderToolSettings**

