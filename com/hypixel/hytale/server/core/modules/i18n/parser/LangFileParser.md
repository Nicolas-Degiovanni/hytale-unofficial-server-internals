---
description: Architectural reference for LangFileParser
---

# LangFileParser

**Package:** com.hypixel.hytale.server.core.modules.i18n.parser
**Type:** Utility

## Definition
```java
// Signature
public class LangFileParser {
```

## Architecture & Concepts
The LangFileParser is a stateless, low-level utility component responsible for transforming a character stream, representing a Hytale language file, into an in-memory key-value map. It serves as the foundational parsing engine for the server's internationalization (i18n) module.

Its sole responsibility is to interpret the `.lang` file format, which includes handling key-value pairs, comments, multi-line values, and basic escape sequences. The design decouples the act of parsing from file I/O; it operates on an abstract `BufferedReader`, allowing it to parse data from any source (files, network streams, in-memory strings) without modification. This component is a pure data transformer, invoked by higher-level services like a `TranslationManager` or `AssetLoader` during the server's resource loading phase.

## Lifecycle & Ownership
- **Creation:** As a static utility class, LangFileParser is never instantiated. Its methods are invoked directly on the class.
- **Scope:** The class and its static methods are available for the entire application lifetime after being loaded by the ClassLoader. It has no instance-specific scope or lifecycle.
- **Destruction:** Not applicable. The class is unloaded upon JVM shutdown.

## Internal State & Concurrency
- **State:** LangFileParser is entirely stateless. It contains no static or instance fields that hold data between invocations. All state required for parsing (e.g., current line number, current multi-line key) is confined to local variables within the `parse` method's stack frame.
- **Thread Safety:** The `parse` method is thread-safe. Multiple threads can invoke it concurrently without risk of interference, **provided that each thread supplies its own unique `BufferedReader` instance**. Sharing a single `BufferedReader` across threads for concurrent parsing is an anti-pattern and will lead to unpredictable behavior, as `BufferedReader` itself is not thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(BufferedReader reader) | Map<String, String> | O(N) | Parses the character stream from the reader into a map of translations. N is the total number of characters in the stream. Throws `IOException` on I/O failure or `TranslationParseException` on syntax errors. |

## Integration Patterns

### Standard Usage
The parser should be invoked by a service responsible for managing language resources. The caller is responsible for creating the `BufferedReader` and handling potential exceptions.

```java
// Typically called from a higher-level I18nService or ResourceManager
Map<String, String> translations;
try (InputStream is = Files.newInputStream(Paths.get("en_us.lang"));
     BufferedReader reader = new BufferedReader(new InputStreamReader(is, StandardCharsets.UTF_8))) {

    translations = LangFileParser.parse(reader);

} catch (LangFileParser.TranslationParseException e) {
    // CRITICAL: A parsing failure indicates a corrupt language file.
    // The server should log this error and potentially fall back to a default language.
    log.error("Failed to parse language file: " + e.getMessage());
} catch (IOException e) {
    // CRITICAL: An I/O error means the language file could not be read.
    log.error("Failed to read language file: " + e.getMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and provides only static methods. Attempting `new LangFileParser()` is a compilation error and violates its design as a stateless utility.
- **Ignoring Exceptions:** The `parse` method throws checked exceptions that signal critical failures. Swallowing or ignoring `TranslationParseException` or `IOException` can result in an incomplete or empty translation map, leading to missing text and runtime `NullPointerException`s throughout the server.
- **Reusing Readers:** Do not attempt to reuse a `BufferedReader` after it has been fully consumed by the `parse` method. A new reader must be created for each parsing operation.

## Data Pipeline
LangFileParser is a single, critical step in the localization data pipeline. It acts as the bridge between raw text assets and a structured, queryable data format.

> Flow:
> .lang File on Disk -> `FileInputStream` -> `BufferedReader` -> **LangFileParser.parse()** -> `Map<String, String>` -> I18n Service Cache

