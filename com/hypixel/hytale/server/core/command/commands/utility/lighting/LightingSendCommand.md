---
description: Architectural reference for LightingSendCommand
---

# LightingSendCommand

**Package:** com.hypixel.hytale.server.core.command.commands.utility.lighting
**Type:** Transient

## Definition
```java
// Signature
public class LightingSendCommand extends AbstractCommandCollection {
```

## Architecture & Concepts
The LightingSendCommand class is a container component within the server's command system. It adheres to the Command and Composite design patterns, acting as a non-executable, grouping command that organizes related sub-commands under a single namespace: *lighting send*.

Its sole architectural function is to provide a unified entry point for server administrators to control the network transmission of lighting data. It achieves this by registering two specialized sub-commands, LightingSendLocalCommand and LightingSendGlobalCommand, which directly manipulate global, static configuration flags within the BlockChunk class.

This class represents a critical control plane interface, exposing low-level world engine toggles to the operator level. The actual logic for toggling the flags is encapsulated within its private inner classes, which leverage a functional approach (via lambdas) to get and set the target static fields on BlockChunk. This design decouples the command definition from the system it controls, though the direct modification of static state represents a significant and deliberate tight coupling.

## Lifecycle & Ownership
- **Creation:** Instantiated once by the server's central CommandRegistry during the server bootstrap phase. The registry scans for and registers all command implementations at startup.
- **Scope:** The object instance is a singleton managed by the CommandRegistry and persists for the entire duration of the server session.
- **Destruction:** The instance is dereferenced and becomes eligible for garbage collection only when the server shuts down and the CommandRegistry is cleared.

## Internal State & Concurrency
- **State:** The LightingSendCommand class itself is stateless and immutable. It contains no instance fields that change after construction. However, its primary function is to mutate *external, shared global state* located in `BlockChunk.SEND_GLOBAL_LIGHTING_DATA` and `BlockChunk.SEND_LOCAL_LIGHTING_DATA`.

- **Thread Safety:** The class instance is inherently thread-safe due to its immutability. The operations it triggers, however, are **not**. The sub-commands perform non-atomic writes to static boolean fields.

    **WARNING:** Direct modification of static fields on a core class like BlockChunk is a significant concurrency risk. While the command system may serialize command execution, the threads responsible for chunk generation and network serialization read these flags. Without proper memory barriers (e.g., `volatile` keyword or atomic wrappers), changes may not be visible to other threads, leading to inconsistent behavior where some chunks are sent with lighting data and others are not.

## API Surface
The public contract is almost entirely defined by its constructor, which is invoked by the command system's registration process.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| LightingSendCommand() | Constructor | O(1) | Instantiates the command and registers its `local` and `global` sub-commands. |

## Integration Patterns

### Standard Usage
This class is not intended for direct invocation by developers. It is designed to be discovered and registered by the server's command system. An administrator interacts with it via the server console.

The *effect* of the command is observed by the world engine, which checks the corresponding static flags during the chunk serialization process.

```java
// Example of how the world engine might use the configured state
// This code would exist within the chunk serialization logic, NOT calling the command.

if (BlockChunk.SEND_GLOBAL_LIGHTING_DATA) {
    packet.writeGlobalLighting(chunk.getGlobalLighting());
}

if (BlockChunk.SEND_LOCAL_LIGHTING_DATA) {
    packet.writeLocalLighting(chunk.getLocalLighting());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new LightingSendCommand()` in application logic. The command system is solely responsible for its lifecycle.
- **Bypassing the Command:** Do not modify the `BlockChunk.SEND_..._LIGHTING_DATA` flags directly from other game systems. This command provides a controlled, auditable, and user-facing interface for these debug and performance-tuning flags. Bypassing it breaks this contract and makes server behavior difficult to diagnose.

## Data Pipeline
This component acts on the control plane, not the data plane. Its execution flow is triggered by user input and results in a state change that affects a separate data pipeline.

> **Control Flow:**
> Server Console Input (`/lighting send global true`) → Command Parser → CommandRegistry Dispatcher → **LightingSendCommand** → LightingSendGlobalCommand → `BlockChunk.SEND_GLOBAL_LIGHTING_DATA = true`

> **Affected Data Pipeline (Chunk Serialization):**
> World Engine → Generate BlockChunk → Serialize Chunk for Network → Read `BlockChunk.SEND_GLOBAL_LIGHTING_DATA` → Conditionally Write Lighting Data → Network Packet → Client

