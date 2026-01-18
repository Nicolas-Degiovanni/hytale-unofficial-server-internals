---
description: Architectural reference for AuthConfig
---

# AuthConfig

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Utility

## Definition
```java
// Signature
public class AuthConfig {
```

## Architecture & Concepts
AuthConfig is a static, non-instantiable utility class that serves as the single source of truth for all server-side authentication and session management constants. It centralizes critical configuration details, decoupling services from hardcoded values such as OAuth 2.0 endpoints, API URLs, client identifiers, and security scopes.

This class acts as a manifest for the server's identity and its relationship with the Hytale authentication backend. Its primary architectural function is to provide immutable, application-wide configuration to components responsible for network communication with Hytale's identity and session services, such as the ServerAuthManager.

The inclusion of the static method getServerAudience introduces a dynamic configuration layer. It allows the server's security audience to be overridden via environment variables, a critical pattern for modern containerized deployments where infrastructure, not the application, defines identity.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its static fields are initialized by the JVM ClassLoader when the class is first referenced by another component. The private static field SERVER_AUDIENCE_OVERRIDE is populated at this time by reading the system environment.
- **Scope:** The class and its static members have an application-wide scope, persisting for the entire lifetime of the JVM process.
- **Destruction:** The class definition is unloaded from memory when the application's ClassLoader is garbage collected, which typically occurs only at JVM shutdown.

## Internal State & Concurrency
- **State:** The class is effectively immutable. All public fields are compile-time constants (`public static final`). The single private field, SERVER_AUDIENCE_OVERRIDE, is written to exactly once during static initialization and is never modified again.
- **Thread Safety:** AuthConfig is inherently thread-safe. All state is constant and globally visible. The getServerAudience method is safe to call from any thread, but its behavior depends on the thread safety of the downstream singleton ServerAuthManager.

**WARNING:** While this class is thread-safe, the correctness of the value returned by getServerAudience depends on the initialization state of ServerAuthManager. Calls made before ServerAuthManager is fully initialized will result in a runtime exception.

## API Surface
The public API consists entirely of static constants and a single static method. The constants define endpoints, client IDs, and scopes for interacting with Hytale's backend services.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getServerAudience() | String | O(1) | Retrieves the server audience identifier. Prioritizes the HYTALE_SERVER_AUDIENCE environment variable, falling back to the active server session ID. |

## Integration Patterns

### Standard Usage
Components should access the static fields directly to configure HTTP clients, build authentication requests, or validate tokens.

```java
// Example of configuring an OAuth client
OAuthClient client = new OAuthClient.Builder()
    .setTokenUrl(AuthConfig.OAUTH_TOKEN_URL)
    .setClientId(AuthConfig.CLIENT_ID)
    .setScopes(AuthConfig.SCOPES)
    .build();

// Example of retrieving the dynamic audience
String audience = AuthConfig.getServerAudience();
jwtVerifier.validate(token, audience);
```

### Anti-Patterns (Do NOT do this)
- **Reflection-based Instantiation:** Attempting to bypass the private constructor via reflection will break the singleton utility pattern and is strictly forbidden.
- **Ignoring Environment Overrides:** Logic that depends on the server audience must *always* call getServerAudience(). Directly accessing ServerAuthManager for the session ID bypasses the critical environment variable override, which can cause catastrophic authentication failures in production environments.
- **Early Access:** Calling getServerAudience before the ServerAuthManager has been initialized by the server's main entry point will lead to a NullPointerException or an illegal state exception.

## Data Pipeline
AuthConfig is not a processing component in a data pipeline; it is a *source* of static configuration data that seeds other components.

> Flow (Configuration Seeding):
> **AuthConfig** (Constants) -> OAuthClientBuilder -> HTTP Request -> Hytale Auth Services

> Flow (Dynamic Audience Resolution):
> System Environment -> **AuthConfig**.getServerAudience() -> JWT Validator -> Authentication Result

