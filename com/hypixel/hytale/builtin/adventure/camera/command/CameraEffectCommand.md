---
description: Architectural reference for CameraEffectCommand
---

# CameraEffectCommand

**Package:** com.hypixel.hytale.builtin.adventure.camera.command
**Type:** Configuration Object

## Definition
```java
// Signature
public class CameraEffectCommand extends AbstractCommandCollection {
```

## Architecture & Concepts

The CameraEffectCommand class serves as the server-side administrative entry point for triggering client camera shake effects. It is not a gameplay system itself, but rather a command-line interface that integrates with underlying engine systems to produce a visual result for a targeted player.

Architecturally, this class follows the **Command Collection** pattern. The primary class, CameraEffectCommand, acts as a namespace for the root command, *camshake*. It does not contain execution logic itself; instead, it registers and exposes subcommands that encapsulate specific functionalities.

This implementation highlights two distinct architectural pathways for achieving a similar outcome:

1.  **System-Driven (DamageCommand):** This subcommand integrates with the core DamageSystems. It constructs a formal Damage event, attaches the desired CameraEffect as metadata, and dispatches it into the standard game logic pipeline. This is the canonical, high-level approach for gameplay-related effects, ensuring that all related systems (e.g., audio, UI, entity state) can react to the damage event consistently.

2.  **Direct Packet Injection (DebugCommand):** This subcommand bypasses the high-level game simulation entirely. It directly accesses the target player's network packet handler and writes a CameraShakePacket. This is a low-level, direct-to-client approach intended solely for debugging, testing, or scenarios where the simulation of a full game event is undesirable.

### Lifecycle & Ownership
-   **Creation:** An instance of CameraEffectCommand is created once by the server's central command registry during the bootstrap sequence. The framework discovers and instantiates all command classes to build the server's command tree.
-   **Scope:** The command object, along with its subcommand instances, is held by the command registry and persists for the entire lifetime of the server session.
-   **Destruction:** The object is de-referenced and becomes eligible for garbage collection only upon server shutdown when the command registry is cleared.

## Internal State & Concurrency
-   **State:** The CameraEffectCommand and its inner classes are effectively immutable and stateless. Their fields are final and serve as static configuration for defining command arguments and structure. All state related to a command's execution is contained within the transient CommandContext object passed to the execute method.

-   **Thread Safety:** The command definition objects are inherently thread-safe due to their immutable nature. However, the execution logic within the `execute` methods is invoked by the server's main thread. Any interaction with the game world, such as dispatching a damage event or accessing an entity's components, is **not thread-safe** and must be performed on the main server thread. The command system framework guarantees this contract.

    **Warning:** Never attempt to invoke a command's execute method from an asynchronous task or a separate thread without synchronizing with the main game loop. Doing so will lead to state corruption and server instability.

## API Surface

The public API for this class is its constructor, which is invoked by the command system framework. The class is not intended for programmatic invocation by other game systems. Its primary interface is the command line.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CameraEffectCommand() | Constructor | O(1) | Initializes the root *camshake* command and registers its subcommands. |

## Integration Patterns

### Standard Usage

This class is designed to be used via the server console or by an in-game entity with sufficient permissions. It is not meant to be referenced or called directly from other code.

**Example 1: Triggering a shake via the damage system**
This command simulates 10 points of *Generic* damage on the nearest player, which in turn triggers the default camera shake associated with that damage type.
```
/camshake damage @p Generic 10
```

**Example 2: Triggering a direct shake for debugging**
This command sends a packet to the nearest player to trigger the *Explosion* camera effect with an intensity of 1.5, bypassing all gameplay systems.
```
/camshake debug @p Explosion 1.5
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not create an instance using `new CameraEffectCommand()`. The object will be inert and disconnected from the server's command registry.
-   **Programmatic Execution:** Avoid obtaining a reference to this class to call its `execute` methods. If you must trigger a command from code, use the central command system's dispatch mechanism, which ensures the full CommandContext is constructed and the execution occurs within the correct lifecycle.

## Data Pipeline

The data flow differs significantly between the two subcommands, illustrating the architectural separation between a system-driven event and a direct network packet injection.

**DamageCommand Flow:**
> Console Input -> Command Parser -> **DamageCommand.execute()** -> Damage Event Instantiation -> DamageSystems.executeDamage() -> Entity State Update -> Client Notification

**DebugCommand Flow:**
> Console Input -> Command Parser -> **DebugCommand.execute()** -> PlayerRef.getPacketHandler() -> CameraShakePacket Construction -> Network Layer -> Client Render Engine

