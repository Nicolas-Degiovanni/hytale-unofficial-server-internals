---
description: Architectural reference for ActionLog
---

# ActionLog

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug
**Type:** Transient

## Definition
```java
// Signature
public class ActionLog extends ActionBase {
```

## Architecture & Concepts
The ActionLog class is a concrete implementation of the ActionBase contract, serving as a leaf node within the server-side NPC Behavior System. Its sole architectural purpose is to provide a diagnostic and debugging mechanism for NPC AI. When an NPC's behavior tree or state machine executes this action, it logs a pre-configured message to the server console.

This component is fundamentally data-driven. The specific text to be logged is not hardcoded but is supplied during instantiation by a corresponding builder, BuilderActionLog. This design allows content creators and developers to inject debug information directly into NPC asset definitions without modifying Java code, facilitating rapid iteration and tracing of complex AI decision-making.

It is a critical tool for observing an NPC's state transitions and action selections in a live environment. The internal logging call is rate-limited to prevent log spam from NPCs that might execute this action on every tick.

### Lifecycle & Ownership
- **Creation:** ActionLog instances are never created directly. They are instantiated by the NPC asset pipeline via a BuilderActionLog during the loading and parsing of an NPC's behavior definition from an asset file. The BuilderSupport object provides necessary context, such as resolving dynamic text values.
- **Scope:** An ActionLog object's lifetime is bound to its containing Role or behavior tree definition. It is part of the shared, immutable NPC template. It is **not** created per-NPC instance; rather, a single ActionLog instance is shared by all NPCs using that specific behavior definition.
- **Destruction:** The object is marked for garbage collection when its parent NPC asset definition is unloaded from memory. This typically occurs during a server shutdown or a hot-reload of game assets.

## Internal State & Concurrency
- **State:** This class is effectively immutable. Its primary state, the `text` field, is declared final and is initialized exclusively in the constructor. It holds no other mutable state.
- **Thread Safety:** The ActionLog object is inherently thread-safe due to its immutable design. The `execute` method can be safely called from any thread. However, the broader NPC system is responsible for ensuring that a single NPC entity's update cycle is processed sequentially on a single thread. The underlying HytaleLogger service is itself a thread-safe system.

## API Surface
The public contract is defined by its parent, ActionBase, and is intended for invocation by the NPC behavior processor, not by general-purpose game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canExecute(...) | boolean | O(1) | Returns true if the action is configured with non-null text. A precondition check before execution. |
| execute(...) | boolean | O(1) | Logs the configured message to the server logs, including the NPC's entity index and role name. Always returns true. |

## Integration Patterns

### Standard Usage
Developers do not invoke ActionLog methods directly. The NPC's behavior processor, typically a Role or a Behavior Tree, is responsible for its execution as part of the server's main update loop. The primary interaction is defining the action in an NPC asset file.

The following conceptual example illustrates how the engine invokes the action.

```java
// Conceptual engine code within an NPC's Role update method
// This code is NOT written by developers using the ActionLog

// Assume 'currentAction' is an instance of ActionLog
if (currentAction.canExecute(entityRef, this, sensorInfo, dt, store)) {
    boolean success = currentAction.execute(entityRef, this, sensorInfo, dt, store);
    // Engine proceeds based on success...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionLog()`. This bypasses the asset loading pipeline and will result in a component that is not correctly configured or integrated into the NPC's behavior. All actions must be defined in asset files.
- **Gameplay Logic:** Do not use ActionLog for critical gameplay notifications or player-facing messages. Its output is rate-limited and directed to server logs, which are not a substitute for the game's event bus. Using it for gameplay can lead to lost events and non-deterministic behavior.

## Data Pipeline
The flow for ActionLog begins with an asset definition and ends with a message in a server log file. It does not transform data but rather acts as a terminal sink for diagnostic information.

> Flow:
> NPC Asset File -> Asset Loader -> BuilderActionLog -> **ActionLog Instance** -> NPC Role Processor -> HytaleLogger -> Server Log File

