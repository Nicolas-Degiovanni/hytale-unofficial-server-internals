---
description: Architectural reference for JsonTypeHandler
---

# JsonTypeHandler

**Package:** com.hypixel.hytale.builtin.asseteditor.assettypehandler
**Type:** Abstract Base Class

## Definition
```java
// Signature
public abstract class JsonTypeHandler extends AssetTypeHandler {
```

## Architecture & Concepts
The JsonTypeHandler is a foundational component within the Asset Editor's framework, designed to standardize the processing of all JSON-based asset files. It embodies the **Template Method Pattern**, providing a fixed, reliable skeleton for loading and validating JSON data while delegating the asset-specific interpretation to concrete subclasses.

Its primary architectural role is to act as a deserialization and validation layer. It abstracts the boilerplate logic of:
1.  Reading raw byte arrays from disk.
2.  Converting bytes to a UTF-8 string.
3.  Parsing the string into a structured BsonDocument.
4.  Gracefully handling common parsing exceptions.

By enforcing this structure, the engine ensures that all JSON-based assets are handled consistently. Concrete implementations, such as a `ModelHandler` or `ParticleEffectHandler`, can focus solely on the business logic of interpreting the parsed BsonDocument, rather than duplicating error-prone parsing code.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of JsonTypeHandler are instantiated by a central registry, likely an `AssetTypeHandlerRegistry`, during the Asset Editor's bootstrap sequence. Each handler is mapped to one or more file extensions.
- **Scope:** An instance of a JsonTypeHandler subclass persists for the entire duration of the Asset Editor session. It is a long-lived, stateless service object.
- **Destruction:** The object is eligible for garbage collection when the Asset Editor is shut down and its associated registry is cleared. No explicit destruction logic is required.

## Internal State & Concurrency
- **State:** This base class is **stateless**. It does not maintain any internal state or cache data between method invocations. Its output is purely a function of its inputs. State management, if required, is the responsibility of concrete subclasses.
- **Thread Safety:** The methods within JsonTypeHandler are inherently **thread-safe** due to their stateless nature. They can be safely invoked from multiple worker threads, provided the input arguments themselves are not being mutated concurrently. The overall thread safety of an asset load operation depends on the implementation of the subclass.

## API Surface
The public contract is designed for framework-level integration, not direct developer invocation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadAsset(path, dataPath, data, ...) | AssetLoadResult | O(N) | Overrides the parent method. Parses raw byte data into a BsonDocument and delegates to `loadAssetFromDocument`. This is the template method. |
| loadAssetFromDocument(...) | abstract AssetLoadResult | O(K) | **Abstract method.** Subclasses MUST implement this to define how a parsed BsonDocument is processed. K depends on subclass logic. |
| isValidData(data) | boolean | O(N) | Performs a fast, non-allocating validation check to determine if the byte data represents a structurally valid JSON document. |

*N = size of the input byte array. K = complexity of the subclass's processing logic.*

## Integration Patterns

### Standard Usage
A developer's primary interaction is to **extend** this class to create a handler for a new, custom JSON-based asset type. The framework will then discover and use this handler automatically.

```java
// Example: A handler for a custom ".myasset.json" file
public class MyAssetHandler extends JsonTypeHandler {

    public MyAssetHandler(AssetEditorAssetType config) {
        super(config);
    }

    @Override
    public AssetLoadResult loadAssetFromDocument(
        AssetPath path,
        Path dataPath,
        BsonDocument document,
        AssetUpdateQuery updateQuery,
        EditorClient editorClient
    ) {
        // 1. Extract semantic data from the parsed document
        String assetName = document.getString("name").getValue();
        int version = document.getInt32("version").getValue();

        // 2. Apply business logic based on the asset data
        if (version < 2) {
            // Handle legacy asset or post a warning
            return AssetLoadResult.ASSETS_UNCHANGED;
        }

        // 3. Update game state or asset registries
        // editorClient.getSomeService().registerMyAsset(assetName);

        return AssetLoadResult.ASSETS_CHANGED;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Overriding loadAsset:** Do not override the primary `loadAsset` method. Doing so bypasses the centralized parsing and error handling logic, defeating the purpose of this base class. Implement `loadAssetFromDocument` instead.
- **Direct Instantiation:** Never create an instance using `new MyAssetHandler()`. Handlers are managed by a central registry and should be retrieved from a service context if needed. Direct instantiation breaks the asset type registration system.
- **Assuming Valid Data:** Do not assume the `BsonDocument` passed to `loadAssetFromDocument` is semantically valid. Always check for the existence of required fields and handle potential `null` or incorrect types.

## Data Pipeline
The flow of data through this component is linear and unidirectional, transforming raw bytes into structured data for domain-specific processing.

> Flow:
> Raw File Bytes (`byte[]`) -> **JsonTypeHandler::loadAsset** -> BSON Parser -> `BsonDocument` -> **Subclass::loadAssetFromDocument** -> Asset Registry / Game State

