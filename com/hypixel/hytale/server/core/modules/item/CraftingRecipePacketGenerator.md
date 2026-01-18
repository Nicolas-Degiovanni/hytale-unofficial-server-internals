---
description: Architectural reference for CraftingRecipePacketGenerator
---

# CraftingRecipePacketGenerator

**Package:** com.hypixel.hytale.server.core.modules.item
**Type:** Transient Utility

## Definition
```java
// Signature
public class CraftingRecipePacketGenerator extends AssetPacketGenerator<String, CraftingRecipe, DefaultAssetMap<String, CraftingRecipe>> {
```

## Architecture & Concepts
The CraftingRecipePacketGenerator is a specialized, stateless transformer responsible for serializing server-side crafting recipe data into network packets for client consumption. It serves as a critical bridge between the server's Asset Management System and the network protocol layer.

This class implements the generic AssetPacketGenerator, providing a concrete strategy for handling the lifecycle of CraftingRecipe assets. Its primary function is to translate changes in the server's asset state—such as initial loading, live updates, or removals—into discrete, well-defined UpdateRecipes packets. This mechanism is fundamental to the server's ability to synchronize game content with connected clients, enabling features like live content reloading without requiring a client restart.

The generator distinguishes between three core synchronization events, each handled by a dedicated method:
1.  **Initialization:** A full dump of all known recipes, sent to a client upon connection.
2.  **Update:** An incremental addition or modification of specific recipes, sent when assets are changed on the server.
3.  **Removal:** A notification that specific recipes are no longer valid and should be deleted from the client's state.

## Lifecycle & Ownership
- **Creation:** An instance of CraftingRecipePacketGenerator is not created directly. It is instantiated by the server's central AssetService during the registration process for the CraftingRecipe asset type. It is part of the configuration that defines how a specific asset type is managed and synchronized.
- **Scope:** The object's lifetime is bound to the registration of the CraftingRecipe asset type within the AssetService. It persists as long as the server is configured to manage and distribute crafting recipes.
- **Destruction:** The instance is eligible for garbage collection when the AssetService is shut down or when the CraftingRecipe asset type is programmatically unregistered from the system.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and its methods operate solely on the parameters provided at invocation. Each method call is an independent, idempotent transformation of input data into an output packet.
- **Thread Safety:** The class is inherently **thread-safe**. Due to its stateless design, multiple threads can safely call its methods concurrently without risk of data corruption or race conditions. The caller is responsible for ensuring the thread safety of the collections passed as arguments.

## API Surface
The public API consists of three override methods that fulfill the AssetPacketGenerator contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| generateInitPacket(assetMap, assets) | Packet | O(N) | Generates a packet containing all specified recipes. Used for initial client synchronization. N is the total number of recipes. |
| generateUpdatePacket(assetMap, loadedAssets, query) | Packet | O(M) | Generates a packet containing only newly added or modified recipes. M is the number of changed recipes. |
| generateRemovePacket(assetMap, removed, query) | Packet | O(K) | Generates a packet containing the keys of recipes to be removed from the client. K is the number of removed recipes. |

## Integration Patterns

### Standard Usage
This class is an internal component of the asset pipeline and should not be invoked directly by typical game logic. The server's AssetService automatically uses this generator when changes to CraftingRecipe assets are detected.

The conceptual flow is managed by the asset system:
```java
// Conceptual example within the AssetService
// This code is NOT for direct use by developers

// 1. AssetService detects a file change and reloads recipes
Map<String, CraftingRecipe> updatedRecipes = assetLoader.loadUpdates(query);

// 2. The service retrieves the registered generator for this asset type
AssetPacketGenerator generator = assetTypeRegistry.getPacketGenerator(CraftingRecipe.class);

// 3. The generator creates the network packet
Packet packet = generator.generateUpdatePacket(assetMap, updatedRecipes, query);

// 4. The packet is broadcast to relevant clients
networkManager.broadcast(packet);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new CraftingRecipePacketGenerator()`. The asset system is responsible for its lifecycle. Manually creating an instance bypasses the entire asset management and registration framework.
- **Manual Packet Construction:** Do not manually create and populate an UpdateRecipes packet. The source of truth for recipes is the Asset System. Modifying assets through the correct channels will automatically trigger this generator, ensuring data consistency between the server state and client state.

## Data Pipeline
The CraftingRecipePacketGenerator is a transformation step within the server's asset-to-client data pipeline.

> Flow:
> Asset File Change -> AssetService Detection -> CraftingRecipe Loader -> **CraftingRecipePacketGenerator** -> UpdateRecipes Packet -> Network System -> Client State Update

