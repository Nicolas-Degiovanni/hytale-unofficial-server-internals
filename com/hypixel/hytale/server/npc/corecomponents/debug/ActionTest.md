---
description: Architectural reference for ActionTest
---

# ActionTest

**Package:** com.hypixel.hytale.server.npc.corecomponents.debug
**Type:** Transient

## Definition
```java
// Signature
public class ActionTest extends ActionBase {
```

## Architecture & Concepts
The ActionTest class is a concrete implementation of the ActionBase contract, designed exclusively for debugging and diagnostic purposes within the server-side NPC Behavior System. An Action represents a leaf node in a behavior tree—a specific, executable task that an NPC can perform.

The primary role of ActionTest is not to implement game logic, but to serve as a validation endpoint for the NPC asset pipeline. When an NPC asset is loaded, its configuration is parsed by a corresponding builder object. ActionTest is constructed by its builder, BuilderActionTest, and immediately logs every data type it receives. This provides developers with a direct, runtime confirmation that various data types—primitives, enums, asset references, and arrays—are being correctly deserialized from asset files and passed into the behavior system.

It is a critical tool for verifying the integrity of the data flow from configuration files to live game objects, but it performs no actions that affect the game world.

## Lifecycle & Ownership
- **Creation:** An ActionTest instance is created by the NPC asset loading system, specifically through its corresponding BuilderActionTest. This occurs when an NPC's behavior tree is being constructed from its asset definition and that definition includes an action of this type.
- **Scope:** The object's lifetime is bound to the lifecycle of the parent NPC entity. It persists as a node within the NPC's in-memory behavior tree for as long as the NPC exists.
- **Destruction:** The instance is marked for garbage collection when the parent NPC is unloaded or destroyed, and its behavior tree is dereferenced. It does not require explicit cleanup.

## Internal State & Concurrency
- **State:** The ActionTest class is effectively **immutable**. All of its operations are performed within the constructor, and it does not define any mutable fields. Its purpose is to process and log its initial configuration, not to maintain state over time.
- **Thread Safety:** This class is **not thread-safe** and must only be accessed from the main server thread that processes NPC logic. The Hytale engine's NPC system operates on a single-threaded model to ensure deterministic behavior. While this specific class has no mutable state, the parent ActionBase and the surrounding behavior tree framework are not designed for concurrent access.

## API Surface
The public contract of ActionTest is limited to its constructor. All other behavior is inherited from ActionBase.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ActionTest(builder, support) | constructor | O(N) | Constructs the action, where N is the number of properties defined in the asset. Immediately logs all configured values to the NPC logger for validation. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly in Java code. Its integration is declarative, specified within an NPC's asset definition file (e.g., JSON or HOCON). The system will then automatically instantiate it during asset loading.

A developer would use this by adding an entry to an NPC's behavior asset to test the data pipeline:

```json
// Hypothetical NPC asset definition
"behavior": {
  "root": {
    "type": "Sequence",
    "children": [
      {
        "type": "ActionTest",
        "comment": "This action will print all its values to the log on load.",
        "booleanValue": true,
        "intValue": 123,
        "stringValue": "Test String",
        "assetValue": "hytale:item_sword"
      }
    ]
  }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new ActionTest()` in game logic. The class is tightly coupled to the asset pipeline and its builder system. Manual construction bypasses the entire data-driven configuration process it is designed to validate.
- **Production Use:** This component must be removed from NPC assets before shipping a final product. It generates significant log spam and serves no gameplay purpose, introducing unnecessary overhead. It is strictly a development and debugging tool.

## Data Pipeline
ActionTest sits at the end of the NPC asset loading pipeline, acting as a final verification step. Its output is directed to the server logging system.

> Flow:
> NPC Asset File -> Server Asset Manager -> BuilderActionTest -> **ActionTest Instance** -> HytaleLogger -> Server Log Output

