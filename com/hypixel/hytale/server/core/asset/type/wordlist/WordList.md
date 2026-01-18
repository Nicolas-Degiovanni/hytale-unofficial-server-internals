---
description: Architectural reference for WordList
---

# WordList

**Package:** com.hypixel.hytale.server.core.asset.type.wordlist
**Type:** Data Asset

## Definition
```java
// Signature
public class WordList implements JsonAssetWithMap<String, DefaultAssetMap<String, WordList>> {
```

## Architecture & Concepts

The WordList class is a data-driven component of the Hytale Asset System, designed to represent a themed collection of translatable terms. It serves as a bridge between abstract data definitions (e.g., a list of animal types in a JSON file) and the server's Internationalization (I18n) framework.

At its core, a WordList is a simple data container. Its primary responsibility is to hold a list of partial translation keys. The true power of the system lies in its integration with the asset loading pipeline. A static `AssetBuilderCodec`, named CODEC, defines the deserialization and transformation logic. When the server loads assets, this codec reads a source file, instantiates a WordList object, and then immediately runs a post-processing step (`processConfig`) to convert the partial keys (e.g., "cow") into fully-qualified translation keys (e.g., "wordlists.animals.cow").

All loaded WordList instances are managed by a static, lazily-initialized `AssetStore`, which acts as a global, in-memory registry. This design ensures that game logic does not need to manage the loading or lifecycle of these assets; it can simply request a WordList by its unique key (e.g., "animals") from a centralized, globally accessible source.

## Lifecycle & Ownership

-   **Creation:** WordList instances are created exclusively by the Hytale `AssetStore` framework during the server's boot sequence or during an asset hot-reload. The static CODEC field is invoked by the asset loader to deserialize a data file (e.g., a JSON file) into a populated WordList object. Direct instantiation by developers is prohibited.
-   **Scope:** An instance of a WordList persists for the entire server session once loaded. It is cached in the static `ASSET_STORE` and shared across all server systems that require it.
-   **Destruction:** Instances are garbage collected when the `AssetRegistry` is cleared, which typically occurs during server shutdown or a full asset reload. There is no manual destruction method.

## Internal State & Concurrency

-   **State:** A WordList object is effectively immutable after its creation and post-processing. Its internal fields, `id` and `translationKeys`, are populated by the asset loader and are not modified during runtime operations. The class itself is stateless.
-   **Thread Safety:** The object's internal state is safe to read from multiple threads due to its immutability. However, methods like `pickTranslationKey` are not inherently thread-safe as they operate on externally provided, mutable arguments (`Random` and `Set`). The caller is responsible for ensuring that access to these arguments, particularly the `alreadyUsedTranslated` set, is properly synchronized if used in a multi-threaded context. The static `getAssetStore` method contains a check-then-act pattern for lazy initialization which, while not technically thread-safe, is benign in practice as it is almost always first called from a single thread during server initialization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWordList(String assetKey) | static WordList | O(1) | Retrieves a loaded WordList from the global registry. Returns a shared EMPTY instance if the key is null or not found. This is the primary entry point for accessing WordLists. |
| pickTranslationKey(Random, Set, String) | String | O(N) | Selects a random, fully-qualified translation key from the list. It filters out any keys whose translated value is already present in the `alreadyUsedTranslated` set. Returns null if no available keys remain. |
| pickDefaultLanguage(Random, Set) | String | O(N) | A convenience method that calls `pickTranslationKey` and immediately resolves the resulting key to a translated string using the default "en-US" language. |

## Integration Patterns

### Standard Usage

The intended use is to retrieve a WordList from the global store via its asset key and then use it to select random, non-repeating translated terms for use in game mechanics.

```java
// How a developer should normally use this
import java.util.Random;
import java.util.HashSet;
import java.util.Set;

// Retrieve the WordList for "animals"
WordList animals = WordList.getWordList("animals");

// Prepare for picking unique words
Random random = new Random();
Set<String> usedWords = new HashSet<>();

// Pick a random animal, translated to the default language
String firstAnimal = animals.pickDefaultLanguage(random, usedWords);
if (firstAnimal != null) {
    // The set now tracks the *translated* word to avoid duplicates
    usedWords.add(firstAnimal);
}

// ... later, pick another unique animal
String secondAnimal = animals.pickDefaultLanguage(random, usedWords);
if (secondAnimal != null) {
    usedWords.add(secondAnimal);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new WordList()`. The constructor is protected and intended for framework use only. All instances must be retrieved via `WordList.getWordList(key)`.
-   **Assuming Existence:** Do not assume a WordList exists. Always check the result of `getWordList` against the static `EMPTY` instance or check if its internal lists are empty before using, as a missing asset file will result in a non-functional object.
-   **Concurrent Modification:** Do not call `pickTranslationKey` from multiple threads with a shared `Set` instance without external synchronization. Doing so will create a race condition and lead to incorrect behavior.

## Data Pipeline

The WordList system involves a two-stage data flow: one for loading and one for runtime usage.

**Loading Pipeline:**
> Flow:
> JSON Asset File on Disk -> `AssetStore` Loader -> **WordList.CODEC** (Deserialization) -> `processConfig()` (Key Transformation) -> In-Memory `WordList` Instance in `ASSET_STORE`

**Runtime Usage Pipeline:**
> Flow:
> Game Logic -> `WordList.getWordList()` -> **WordList.pickTranslationKey()** -> `I18nModule.getMessage()` -> Final Translated String

---

