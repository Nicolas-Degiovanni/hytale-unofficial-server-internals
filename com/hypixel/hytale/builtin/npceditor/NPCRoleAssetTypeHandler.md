---
description: Architectural reference for NPCRoleAssetTypeHandler
---

# NPCRoleAssetTypeHandler

**Package:** com.hypixel.hytale.builtin.npceditor
**Type:** Transient

## Definition
```java
// Signature
public class NPCRoleAssetTypeHandler extends JsonTypeHandler {
```

## Architecture & Concepts
The NPCRoleAssetTypeHandler is a specialized component within the server-side Asset Editor framework. It acts as a concrete implementation of the strategy pattern, providing specific logic for handling assets of type **NPCRole**.

This class serves as a critical bridge between the generic, file-agnostic Asset Editor system and the domain-specific NPCPlugin. When the Asset Editor detects a file system event (creation, modification, deletion) for a JSON file designated as an NPCRole, it delegates the processing of that event to this handler.

Its primary responsibility is to translate high-level asset lifecycle events into direct, state-changing calls on the NPCPlugin's BuilderManager. By extending JsonTypeHandler, it inherits the capability to process BSON documents parsed from JSON files, but its core function is orchestration, not data transformation.

## Lifecycle & Ownership
- **Creation:** An instance of NPCRoleAssetTypeHandler is created and registered by the Asset Editor framework during server initialization. The framework discovers all AssetTypeHandler implementations and maps them to their declared asset type IDs.
- **Scope:** The instance persists for the entire server session. It is held in a central registry within the Asset Editor system and is invoked whenever an asset of type NPCRole is manipulated.
- **Destruction:** The object is eligible for garbage collection upon server shutdown when the Asset Editor's handler registry is cleared.

## Internal State & Concurrency
- **State:** This class is entirely stateless. It contains no instance fields and does not cache any data. Its behavior is purely a function of the arguments passed to its methods and the state of the singleton services it interacts with, such as NPCPlugin and AssetModule.
- **Thread Safety:** The class is inherently thread-safe due to its stateless nature. However, it is designed to be invoked by the Asset Editor's single-threaded event processing loop. The methods it calls, particularly on the NPCPlugin's BuilderManager, are responsible for ensuring their own thread safety, as they manipulate core game state.

## API Surface
The public API is a contract defined by the AssetTypeHandler interface, intended for consumption by the Asset Editor framework exclusively.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadAssetFromDocument(...) | AssetLoadResult | O(N) | Invoked when an NPCRole asset is created or modified. Delegates to the NPCPlugin's BuilderManager to load the file. Complexity depends on the asset's contents. |
| unloadAsset(...) | AssetLoadResult | O(N) | Invoked when an NPCRole asset is deleted. Delegates to the NPCPlugin's BuilderManager to remove the corresponding in-memory definition. |
| restoreOriginalAsset(...) | AssetLoadResult | O(N) | Invoked when an asset is reverted. Delegates to the NPCPlugin's BuilderManager to reload the original file from the asset pack. |
| getDefaultUpdateQuery() | AssetUpdateQuery | O(1) | Returns a default configuration instructing the Asset Editor framework on how to handle subsequent processing, in this case, to avoid a full asset rebuild. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by game logic or plugin developers. It is a framework component, automatically invoked by the Asset Editor. A developer creating a new custom asset type would implement a similar handler and register it.

The conceptual interaction is as follows:

```java
// Pseudo-code for how the Asset Editor Framework uses this handler

// 1. Framework detects a file change for "my_role.json"
AssetPath assetPath = new AssetPath("pack1", "npcs/roles/my_role.json");
AssetTypeHandler handler = assetTypeRegistry.getHandlerFor("NPCRole"); // Returns NPCRoleAssetTypeHandler instance

// 2. Framework reads the file and calls the handler
BsonDocument document = readJsonFile(assetPath);
handler.loadAssetFromDocument(assetPath, dataPath, document, query, client);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new NPCRoleAssetTypeHandler()`. The framework is responsible for its creation and registration. Manually creating an instance will have no effect on the system.
- **Direct Invocation:** Do not call methods like `loadAssetFromDocument` directly. This bypasses the Asset Editor's state management and file-watching logic, which can lead to a desynchronized state between the file system and the running server. Asset changes must always be initiated through file system modifications that the editor can detect.

## Data Pipeline
This handler acts as a specific processing node in the Asset Editor's data flow. It receives data from the framework and forwards commands to the NPC system.

> Flow:
> File System Event (e.g., file save) -> Asset Editor Watcher -> Asset Type Identification ("NPCRole") -> **NPCRoleAssetTypeHandler** -> NPCPlugin.BuilderManager -> In-Memory NPC Role Cache Update

