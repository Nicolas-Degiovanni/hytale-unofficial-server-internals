---
description: Architectural reference for TeleporterSettingsPageSupplier
---

# TeleporterSettingsPageSupplier

**Package:** com.hypixel.hytale.builtin.adventure.teleporter.page
**Type:** Transient Factory

## Definition
```java
// Signature
public class TeleporterSettingsPageSupplier implements OpenCustomUIInteraction.CustomPageSupplier {
```

## Architecture & Concepts
The TeleporterSettingsPageSupplier is a data-driven factory responsible for instantiating a TeleporterSettingsPage. It acts as a crucial bridge between the server's Interaction System and the UI framework. Its primary role is to define the logic for what happens when a player interacts with a block that is meant to open the teleporter configuration screen.

This class is not a long-lived service. Instead, instances are created by the engine's Codec system when parsing game asset files, typically a block's JSON or HOCON definition. This allows designers and developers to declaratively specify that a block should open a UI, and to configure the behavior of that UI without writing Java code.

A key architectural feature is its ability to perform **lazy initialization of block entities**. If a player interacts with a block that has no associated Teleporter component, this supplier can be configured to create and attach the component on-the-fly. This avoids the overhead of pre-populating the world with teleporter entities that may never be used.

## Lifecycle & Ownership
- **Creation:** Instantiated by the Hytale Codec system during asset loading. An instance is created based on configuration found within a game data file, such as a block definition that includes an OpenCustomUIInteraction. It is **not** intended to be manually instantiated in code.
- **Scope:** Short-lived and stateless. An instance is held by an OpenCustomUIInteraction configuration. Its lifetime is effectively bound to the processing of that single interaction.
- **Destruction:** The object is eligible for garbage collection immediately after the `tryCreate` method is invoked and the interaction event has been fully processed. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
- **State:** Effectively immutable. Its internal fields—*create*, *mode*, and *activeState*—are populated once by the Codec during deserialization and are not modified thereafter. The supplier itself does not cache or manage any runtime game state.
- **Thread Safety:** The object's immutable nature makes it inherently thread-safe. However, the `tryCreate` method performs world-state modifications, such as creating new entities. These operations are **not thread-safe** and must be executed exclusively on the main server thread. The engine's Interaction System guarantees this contract.

**WARNING:** Calling `tryCreate` from any thread other than the main world tick thread will lead to race conditions, chunk corruption, and server instability.

## API Surface
The public contract is minimal, exposing only the factory method required by the CustomPageSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tryCreate(ref, accessor, playerRef, context) | CustomUIPage | Variable | Factory method to create a TeleporterSettingsPage. Returns null if the interaction context is invalid (e.g., no target block). Complexity depends on chunk cache state and whether a new block entity must be created. |

## Integration Patterns

### Standard Usage
The TeleporterSettingsPageSupplier is not used imperatively in Java code. Instead, it is configured declaratively within a game asset file. The engine's interaction system resolves and invokes it automatically.

Below is a conceptual example of how it would be defined in a block's configuration file.

```json
// Example: my_teleporter_block.json
{
  "interactions": [
    {
      "type": "hytale:open_custom_ui",
      "pageSupplier": {
        "type": "hytale:teleporter_settings_page",
        "Create": true,
        "Mode": "FULL",
        "ActiveState": "default"
      }
    }
  ]
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new TeleporterSettingsPageSupplier()`. This bypasses the critical data-driven configuration intended by the Codec system, resulting in a supplier with default and likely incorrect behavior.
- **Stateful Implementation:** Do not extend this class to hold temporary state between calls. Each invocation of `tryCreate` should be treated as an independent, atomic operation.
- **Off-Thread Execution:** Never invoke the `tryCreate` method from a separate thread. All world interactions must be synchronized with the main server tick.

## Data Pipeline
The supplier sits in the middle of a data flow that begins with a player action and ends with a UI update on the client.

> Flow:
> Player Interaction Event -> Interaction System -> **TeleporterSettingsPageSupplier.tryCreate()** -> New Teleporter Entity (optional) -> TeleporterSettingsPage Instance -> UI System -> Client Render

