---
description: Architectural reference for GithubMessageUtil
---

# GithubMessageUtil

**Package:** com.hypixel.hytale.logger.util
**Type:** Utility

## Definition
```java
// Signature
public class GithubMessageUtil {
```

## Architecture & Concepts
The GithubMessageUtil class is a stateless, special-purpose formatter designed to bridge the Hytale logging infrastructure with the GitHub Actions continuous integration environment. Its sole responsibility is to translate internal error or warning data into a specific string format that the GitHub Actions runner can parse.

When these specially formatted strings are printed to standard output during a CI build, the GitHub user interface automatically renders them as code annotations. This provides developers with direct feedback on pull requests and commits, highlighting issues directly on the relevant lines of code.

This class acts as an adapter, decoupling the core application from the proprietary format of a specific CI/CD system. It ensures that logging calls within the engine do not need to be aware of the external environment they are running in.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a utility class composed entirely of static methods and a static final field, its lifecycle is managed by the JVM ClassLoader. It is loaded into memory on the first access to any of its static members.
- **Scope:** The class has a static scope and persists for the entire lifetime of the Java Virtual Machine.
- **Destruction:** The class and its static members are unloaded from memory when the JVM shuts down. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** GithubMessageUtil is entirely stateless. The private field CI is initialized once from a system environment variable at class-loading time and is declared `static final`, making it an immutable constant for the duration of the application's execution. The methods are pure functions, producing output based only on their input arguments.
- **Thread Safety:** This class is inherently thread-safe. Its stateless nature and lack of side effects guarantee that all methods can be invoked concurrently from any number of threads without synchronization or risk of data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isGithub() | boolean | O(1) | Returns true if the CI environment variable is set, indicating the code is likely running within a GitHub Actions workflow. |
| messageError(file, line, col, msg) | String | O(N) | Formats a message string as a GitHub Actions error annotation, pointing to a specific file, line, and column. N is the length of the message. |
| messageError(file, msg) | String | O(N) | Formats a message string as a file-level GitHub Actions error annotation. |
| messageWarning(file, line, col, msg) | String | O(N) | Formats a message string as a GitHub Actions warning annotation, pointing to a specific file, line, and column. |
| messageWarning(file, msg) | String | O(N) | Formats a message string as a file-level GitHub Actions warning annotation. |

## Integration Patterns

### Standard Usage
This utility should be used within build scripts, custom log appenders, or other CI-aware tooling. The primary pattern is to first check the environment and then format and print the message.

```java
// Example from a custom logging system or build tool
if (GithubMessageUtil.isGithub()) {
    String formattedError = GithubMessageUtil.messageError("src/Game.java", 42, 5, "Null pointer risk detected");
    System.out.println(formattedError);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and is not designed to be instantiated. All access must be through its static methods. Attempting to instantiate it is a design violation.
- **Unconditional Printing:** Do not call the formatting methods and print their output without first checking `GithubMessageUtil.isGithub()`. Doing so will pollute local development console logs with unreadable, environment-specific strings.

## Data Pipeline
The class functions as a simple, single-stage transformation pipeline for logging data. It does not initiate I/O but prepares data for an I/O consumer.

> Flow:
> Internal Log Event -> **GithubMessageUtil** -> Formatted Annotation String -> Standard Output -> GitHub Actions Runner -> GitHub UI Annotation Display

