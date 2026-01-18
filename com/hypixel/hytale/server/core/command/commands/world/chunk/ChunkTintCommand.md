---
description: Architectural reference for ChunkTintCommand
---

# ChunkTintCommand

**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient Handler

## Definition
```java
// Signature
public class ChunkTintCommand extends AbstractPlayerCommand {
```

## Architecture & Concepts
The ChunkTintCommand is a server-side administrative command that provides a direct interface for modifying the visual tint of world chunks. It is a concrete implementation of the AbstractPlayerCommand, integrating it into the server's core command processing system. This command is designed for developers, content creators, and administrators to apply color overlays to terrain for aesthetic or debugging purposes.

Architecturally, this class serves as a high-level entry point that translates a player's text-based request into low-level world data manipulation. It directly interacts with the core world representation, specifically the ChunkStore and its underlying BlockChunk components.

The command features two primary operational modes:
1.  **Direct Tinting:** Applies a single, solid color to all blocks within the player's current chunk.
2.  **Gaussian Blur Tinting:** Applies a more computationally intensive tint that blends colors across a specified radius of chunks using a Gaussian blur algorithm. This creates smooth color transitions across the terrain.

This command also exposes a sub-command that integrates with the server's custom UI system, demonstrating a powerful pattern where complex command arguments can be provided through a graphical interface instead of text flags.

## Lifecycle & Ownership
-   **Creation:** A single instance of ChunkTintCommand is instantiated by the CommandManager or a related service during the server's bootstrap phase. It is registered under the `chunk tint` command path.
-   **Scope:** The command object is long-lived, persisting for the entire duration of the server session. It is designed to be stateless between executions.
-   **Destruction:** The instance is dereferenced and eligible for garbage collection only when the server shuts down and the CommandManager clears its command registry.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. The argument definitions (colorArg, radiusArg, etc.) are immutable configurations set during construction. All state required for execution, such as the player's position, world reference, and command arguments, is passed into the `execute` method via the CommandContext. All collections and maps used for processing are created locally within the method scope.
-   **Thread Safety:** This command is **not thread-safe**. It performs direct, unsynchronized writes to BlockChunk components. It is fundamentally designed to be executed serially by the server's main logic thread or a dedicated, single-threaded command processor. Invoking the `execute` method concurrently would lead to severe data corruption and server instability. The engine's CommandManager is responsible for ensuring serialized execution.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| execute(...) | protected void | O(1) to O(R²) | The primary entry point for command execution. Complexity is O(1) for a simple tint and O(R²) for a blur, where R is the blur radius in blocks. |

## Integration Patterns

### Standard Usage
This command is intended to be run by a player or the server console. The CommandManager handles parsing and invocation.

```java
// Example command strings invoked by a player
// Apply a solid red tint to the current chunk
/chunk tint #FF0000

// Apply a blurred green tint with a 10-block radius
/chunk tint #00FF00 --blur --radius=10

// Open the interactive UI for tinting
/chunk tint
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ChunkTintCommand()` in game logic. The command system requires commands to be registered at startup. Creating new instances has no effect.
-   **Manual Execution:** Avoid calling the `execute` method directly. This bypasses critical infrastructure like permission checks, argument parsing, and thread safety guarantees provided by the CommandManager. Always use `CommandManager.get().handleCommand(...)`.
-   **Stateful Implementation:** Do not add mutable instance fields to this class. The command handler must remain stateless to ensure predictable behavior across multiple executions by different players.

## Data Pipeline
The flow of data through this command is critical to understanding its impact on the world state.

> **Simple Tint Flow:**
> Player Input (`/chunk tint #...`) -> CommandManager Parser -> **ChunkTintCommand.execute** -> Get Player TransformComponent -> Calculate Chunk Index -> World.getChunkStore -> Get BlockChunk Component -> Loop 32x32 & call `blockChunk.setTint` -> World.getNotificationHandler().updateChunk -> Network Packet to Clients

> **Blur Tint Flow:**
> Player Input (`/chunk tint #... --blur`) -> CommandManager Parser -> **ChunkTintCommand.execute** -> Create LocalCachedChunkAccessor for region -> Iterate over a grid (32 + 2*radius)² -> For each block, call `blur()` helper -> `blur()` reads tints from neighboring chunks via accessor -> Calculate weighted average color -> Store results in a temporary Long2IntOpenHashMap -> Iterate map and apply new tints to multiple BlockChunk components -> World.getNotificationHandler().updateChunk for all affected chunks -> Multiple Network Packets to Clients

---

# Nested Components

## TintChunkPageCommand
**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient Handler

### Architecture & Concepts
This nested class is a lightweight command registered as a usage variant of the main ChunkTintCommand. Its sole responsibility is to open the TintChunkPage custom UI for the executing player. It acts as a bridge between the text-based command system and the interactive UI framework.

### Standard Usage
When a player executes `/chunk tint` with no arguments, the CommandManager resolves to this variant.

```java
// Inside the command system
// This command is invoked, which then opens a UI for the player.
playerComponent.getPageManager().openCustomPage(ref, store, new ChunkTintCommand.TintChunkPage(playerRef));
```

## TintChunkPage
**Package:** com.hypixel.hytale.server.core.command.commands.world.chunk
**Type:** Transient UI Page

### Architecture & Concepts
TintChunkPage is an implementation of InteractiveCustomUIPage. It represents a server-authoritative user interface that is sent to the client for rendering.

Its lifecycle is tied to a specific player's interaction. When created, its `build` method is called to define the UI layout file (`Pages/TintChunkPage.ui`) and establish event bindings. These bindings link client-side UI actions (e.g., clicking a button, changing a color picker value) to server-side logic.

When a player interacts with the UI, the client sends an event packet to the server. This packet is deserialized into a TintChunkPageEventData object using a pre-defined Codec. The `handleDataEvent` method is then invoked with this data, allowing the server to react by executing the final tint command or updating the UI state.

This component exemplifies the server-driven UI pattern, where the server retains all logic and state, and the client is primarily a rendering and input-forwarding engine.

### Data Pipeline

> **UI Event Flow:**
> Client UI Interaction (e.g., Button Click) -> Network Packet with UI Payload -> Server Packet Handler -> **TintChunkPage.handleDataEvent** -> Deserialize payload into TintChunkPageEventData -> `CommandManager.get().handleCommand(...)` -> World modification as described above.

