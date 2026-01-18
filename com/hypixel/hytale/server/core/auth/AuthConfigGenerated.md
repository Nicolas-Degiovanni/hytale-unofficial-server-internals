---
description: Architectural reference for AuthConfigGenerated
---

# AuthConfigGenerated

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Utility

## Definition
```java
// Signature
final class AuthConfigGenerated {
```

## Architecture & Concepts
The AuthConfigGenerated class serves as a centralized, compile-time provider of constants for the authentication system. Its primary architectural role is to decouple environment-specific configuration from the core application logic.

The suffix *Generated* strongly implies that this file is not meant to be manually edited. It is produced by an automated build process, which injects values based on the target build environment (e.g., development, staging, release). This pattern ensures that the correct authentication endpoints and domains are used for each deployment without requiring code changes or runtime configuration files. It acts as an immutable, statically-linked source of truth for authentication parameters.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its private constructor explicitly prevents the creation of objects. The class itself is loaded into the JVM by the ClassLoader when it is first referenced by another component, typically an authentication service.
- **Scope:** The class and its static fields persist for the entire lifetime of the server application once loaded.
- **Destruction:** The class is unloaded when the JVM shuts down. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The state is immutable. It consists solely of static final fields, which are compile-time constants. The values are embedded into the bytecode and cannot be changed at runtime.
- **Thread Safety:** This class is inherently thread-safe. As a holder of immutable constants, it can be accessed from any thread without synchronization. No locking mechanisms are necessary.

## API Surface
The public contract consists exclusively of static fields.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DOMAIN | String | O(1) | The root domain for authentication services. |
| ENVIRONMENT | String | O(1) | The name of the deployment environment, e.g., "release". |

## Integration Patterns

### Standard Usage
Components requiring authentication parameters should access the static fields directly. This is the sole intended use case.

```java
// Correctly retrieving the authentication domain
String authDomain = AuthConfigGenerated.DOMAIN;
String url = "https://api." + authDomain + "/auth";
```

### Anti-Patterns (Do NOT do this)
- **Hardcoding Values:** Do not duplicate the constants from this class elsewhere in the codebase. This defeats the purpose of centralized, generated configuration and creates a high risk of environment mismatches.
- **Reflection:** Do not attempt to use reflection to instantiate this class or modify its final fields. Such actions violate its design contract and will lead to undefined behavior.

## Data Pipeline
This class is a data source, not a processing component. It injects static configuration data into other systems at the beginning of a data flow.

> Flow:
> **AuthConfigGenerated** -> Authentication Service -> Outbound Network Request

