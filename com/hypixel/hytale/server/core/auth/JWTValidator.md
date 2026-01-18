---
description: Architectural reference for JWTValidator
---

# JWTValidator

**Package:** com.hypixel.hytale.server.core.auth
**Type:** Transient

## Definition
```java
// Signature
public class JWTValidator {
```

## Architecture & Concepts
The JWTValidator is a security-critical component responsible for validating JSON Web Tokens presented by clients during the authentication process. It acts as a gatekeeper, ensuring that only clients with valid, unexpired, and correctly signed tokens can access protected server resources.

This class is designed to operate against a centralized authentication authority, represented by the SessionServiceClient. Its core responsibility is to verify the cryptographic signature of incoming tokens against a set of public keys (JSON Web Key Set, or JWKS) fetched from this service. This decouples the game server from the key management of the authentication service, allowing for seamless security operations like key rotation.

Key architectural characteristics include:
- **Algorithm Enforcement:** The validator is hardcoded to accept only the EdDSA (JWSAlgorithm.EdDSA) signature algorithm, a modern and secure elliptic curve algorithm. Tokens signed with any other algorithm are rejected immediately.
- **Claim Validation:** Beyond signature verification, it performs a comprehensive check of standard JWT claims, including issuer (iss), audience (aud), expiration (exp), and not-before (nbf). It also validates Hytale-specific claims like username and IP address.
- **Certificate Binding:** For certain token types, it validates a certificate fingerprint claim (x5t#S256) against the client's TLS certificate. This provides an additional layer of security, binding the token to a specific client connection and mitigating token theft.
- **JWKS Caching:** To minimize latency and reduce load on the SessionServiceClient, the validator maintains an in-memory, time-based cache of the JWKS. This is a critical performance feature.

## Lifecycle & Ownership
- **Creation:** An instance of JWTValidator is created during server bootstrap. It is not a global singleton and must be instantiated with its required dependencies: a SessionServiceClient instance and the expected issuer and audience strings for token validation. This is typically handled by a dependency injection framework.
- **Scope:** The object is long-lived, designed to persist for the entire lifetime of the server process. Its internal state, particularly the JWKS cache, provides significant performance benefits that would be lost if it were short-lived.
- **Destruction:** The object has no explicit destruction or cleanup method. It is garbage collected when the server shuts down and its owning context is destroyed.

## Internal State & Concurrency
- **State:** The JWTValidator is stateful and maintains a mutable internal cache. The primary state consists of:
    - cachedJwkSet: The most recently fetched JSON Web Key Set.
    - jwksCacheExpiry: A timestamp indicating when the cache is considered stale.
    - jwksFetchLock: A ReentrantLock to serialize access to the cache during refresh operations.
    - pendingFetch: A CompletableFuture representing an in-flight request to the SessionServiceClient.

- **Thread Safety:** This class is thread-safe and designed for high-concurrency environments.
    - All cache-related fields are declared as volatile to ensure visibility across threads.
    - A ReentrantLock (jwksFetchLock) is used to prevent race conditions during cache updates.
    - The implementation uses a sophisticated non-blocking pattern with a CompletableFuture to manage concurrent requests for a JWKS refresh. If one thread initiates a fetch, subsequent threads will wait on the same future instead of initiating redundant network calls, effectively mitigating the "thundering herd" problem.

    **WARNING:** Direct manipulation of the internal state from outside this class will break its concurrency guarantees and lead to severe security vulnerabilities.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| validateToken(accessToken, clientCert) | JWTClaims | O(1) (cache hit) | Validates a primary client access token. Performs full signature, claim, and certificate binding validation. Returns null on any failure. |
| validateIdentityToken(identityToken) | IdentityTokenClaims | O(1) (cache hit) | Validates a user identity token. Performs signature and claim validation. Returns null on any failure. |
| validateSessionToken(sessionToken) | SessionTokenClaims | O(1) (cache hit) | Validates a short-lived session token. Performs signature and claim validation. Returns null on any failure. |
| invalidateJwksCache() | void | O(1) | Forcibly purges the internal JWKS cache. This will cause the next validation request to trigger a network fetch. |

## Integration Patterns

### Standard Usage
The JWTValidator should be injected as a dependency into network handlers or authentication services. A single, shared instance should be used for all validation operations.

```java
// In an authentication handler or service
public class AuthenticationService {
    private final JWTValidator jwtValidator;

    // Constructor for dependency injection
    public AuthenticationService(JWTValidator jwtValidator) {
        this.jwtValidator = jwtValidator;
    }

    public boolean authenticate(String token, X509Certificate cert) {
        JWTValidator.JWTClaims claims = jwtValidator.validateToken(token, cert);
        
        // A null return indicates validation failure. The request MUST be rejected.
        if (claims == null) {
            return false;
        }

        // Proceed with authenticated user context
        // UUID userUUID = claims.getSubjectAsUUID();
        return true;
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation Per Request:** Do not use `new JWTValidator(...)` for each incoming request. This defeats the entire caching mechanism, will cause extreme performance degradation, and may trigger rate limiting from the SessionServiceClient.
- **Ignoring Null Return:** A null return from any `validate` method is a definitive signal of failure. The client connection must be treated as unauthenticated. Proceeding after a null return is a critical security flaw.
- **Excessive Cache Invalidation:** Calling invalidateJwksCache frequently in a production environment is harmful. It should only be used for specific administrative tasks or in response to a known key compromise event.

## Data Pipeline
The flow of data for a standard access token validation is as follows. The JWTValidator is the central processing component.

> Flow:
> Client Request with JWT -> Network Layer -> **JWTValidator**.validateToken() -> [Internal: JWKS Cache Check -> SessionServiceClient Fetch (if cache miss)] -> Signature & Claim Verification -> Return Claims or Null -> Authentication Service Logic

