# Security and Rate Limiting


## Purpose

The **Security and Rate Limiting** layer protects the EMS from unauthorized access, malicious commands, and accidental overload. It ensures that:

- Only authenticated and authorized users can send commands.
- Edge devices are properly identified and trusted.
- API endpoints and WebSocket connections are protected against brute‑force, denial‑of‑service, and replay attacks.
- System resources are fairly distributed across users and edges.

This layer is implemented across both cloud and edge components, with defense in depth as a guiding principle.

---

## Role in the Command Dispatcher Architecture

Security and rate limiting are cross‑cutting concerns that affect every component of the command dispatcher.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
│  (Next.js, mobile app)                       │
└───────────────┬─────────────────────────────┘
                │ WSS + JWT
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
├─────────────────────────────────────────────┤
│  • TLS termination (WSS/HTTPS)               │
│  • JWT validation (on connect & per command) │
│  • Rate limiting per user/IP/edge            │
│  • Input sanitization                         │
│  • CORS policies                              │
│  • Replay protection (timestamp + cache)     │
│  • Active session revocation (Redis pub/sub) │
└───────────────┬─────────────────────────────┘
                │ WSS + API Key
┌───────────────▼─────────────────────────────┐
│     Edge‑Side Dispatcher                     │
├─────────────────────────────────────────────┤
│  • TLS client certificate (optional)         │
│  • API key validation                         │
│  • Input validation (second layer)           │
│  • Local rate limiting (acknowledgment bursts)│
└─────────────────────────────────────────────┘
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Authentication** | Verifying the identity of a user or edge device. Users use JWT (via Django); edges use API keys. |
| **Authorization** | Determining whether an authenticated entity has permission to perform a specific action (e.g., `can_control` for a specific edge). |
| **Rate Limiting** | Restricting the number of requests or commands a user/edge can issue within a time window to prevent abuse. |
| **TLS (Transport Layer Security)** | Encrypts all communication between UI ↔ cloud and cloud ↔ edge. |
| **Input Sanitization** | Validating and cleaning incoming data to prevent injection attacks. |
| **CORS (Cross-Origin Resource Sharing)** | Controls which web origins can access the cloud APIs. |
| **API Key** | A secret token used by edge devices to authenticate with the cloud. |
| **JWT (JSON Web Token)** | Used for user authentication; issued by Django and signed with a private key. |
| **Replay Attack Prevention** | Ensuring that captured valid requests cannot be resent later. Achieved via timestamps and a short‑lived cache of seen `command_id`s. |
| **Session Revocation** | The ability to terminate active WebSocket sessions when a user's permissions change or account is disabled. |

---

## Functional Requirements

### Authentication

-   All cloud WebSocket and HTTP endpoints shall require TLS (WSS/HTTPS) with minimum TLS 1.3.
    
-   User connections to cloud shall authenticate using a JWT obtained from Django. The JWT shall be included either as a query parameter (`?token=...`) or as a first-message authentication frame.
    
-   Edge connections to cloud shall authenticate using a pre-shared API key. The key shall be sent as a first-message authentication frame:  
    `{"action": "authenticate", "edge_id": "...", "token": "..."}`.
    
-   The cloud shall validate JWT signatures against Django's public key and check token expiry at connection time. Additionally, for every subsequent command received over the WebSocket, the cloud shall re-check that the original JWT has not expired (i.e., the token's `exp` claim is still in the future). If expired, the cloud shall close the WebSocket with close code 4001 (Unauthorized).
    
-   The cloud shall validate edge API keys against a secure store (e.g., hashed in database). Keys shall be revocable and rotatable without downtime.
    
-   Failed authentication attempts shall be logged and may trigger rate limiting or temporary IP bans.


### Authorization

-   After authentication, the cloud shall verify that the user has the `can_control` permission for the target edge/site before forwarding any command.
    
-   Authorization checks shall be performed for every command, not just at connection time (per-command authorization).
    
-   Edge devices shall only be allowed to send commands to themselves; they cannot impersonate other edges.
    

----------

### Rate Limiting

-   The cloud shall enforce rate limits with different scopes:
    
    -   **User-to-Cloud commands:** 10 commands per second per user (applies to command initiation).
        
    -   **Edge-to-Cloud acknowledgments:** 100 messages per second per edge (to accommodate bursts from bulk commands).
        
    -   **IP-based limiting for HTTP endpoints:** 100 requests per minute per IP.
        
-   Rate limit violations shall return HTTP 429 (Too Many Requests) for REST APIs and a `rate_limited` error message over WebSocket.
    
-   Rate limit counters shall be stored in Redis with appropriate TTLs (e.g., sliding window or token bucket).
    
-   The edge may implement local rate limiting to prevent internal flooding, but cloud-side limits are primary.
    

----------

### Input Validation and Sanitization

-   All input (JSON payloads, query parameters) shall be validated against strict schemas. Unexpected fields shall be rejected.
    
-   String fields shall be checked for maximum length to prevent buffer overflows.
    
-   Numeric fields shall be within reasonable ranges (additional to device-specific limits).
    

----------

### Replay Attack Prevention

-   Each command shall include a `command_id` (UUID) and a `timestamp` (ISO 8601 with millisecond precision).
    
-   Upon receiving a command, the cloud shall:
    
    1.  **Timestamp check:** Compute the absolute difference between the server's current time and the payload's `timestamp`. If the difference exceeds a threshold (e.g., 60 seconds), reject the command with `reason: "timestamp_too_old"`.
        
    2.  **Replay cache check:** If the timestamp is within the window, check if the `command_id` exists in a Redis cache with TTL of 60 seconds. If present, reject with `reason: "duplicate_command"`.
        
    3.  If both checks pass, add the `command_id` to the Redis cache (set with 60-second TTL) and proceed with processing.
        

----------

### Session Revocation

-   The cloud shall support active session revocation. Django shall publish a message to a Redis pub/sub channel (e.g., `auth:revoked_users`) whenever a user's account is disabled, permissions are revoked, or roles change.
    
-   All cloud dispatcher instances shall subscribe to this channel. Upon receiving a revocation message for a user, they shall immediately close all active WebSocket connections associated with that user, sending a close code 4001 (Unauthorized).
    

----------

### CORS (Cross-Origin Resource Sharing)

-   The cloud API shall implement strict CORS policies, allowing only configured origins (e.g., production dashboard domain) to access endpoints.
    

----------

### Audit Logging

-   All authentication attempts (success/failure), authorization failures, rate limit hits, and commands shall be logged with timestamps, IP addresses, user/edge IDs, and command IDs.
---

## Detailed Flow

### User Authentication and Command Flow

1. User logs into Next.js dashboard via Django authentication (separate flow).
2. Dashboard obtains a JWT from Django.
3. Dashboard connects to `wss://api.ems.example/v1/commands/ws?token=<jwt>` (or sends auth frame as first message).
4. Cloud validates JWT signature, expiry, and extracts user ID and permissions. It stores the token's expiry time associated with the connection.
5. For each subsequent command:
   - Cloud checks if the original JWT has expired (current time > token expiry). If expired, closes WebSocket with code 4001.
   - Cloud verifies that user still has `can_control` permission for target edge (may be cached for short period).
   - Cloud performs replay checks (timestamp + `command_id` cache).
   - Cloud enforces rate limits (user + edge).
   - Passes command to validation and routing.

### Edge Authentication and Command Reception

1. Edge device starts, loads its `edge_id` and `api_key` from secure storage (e.g., encrypted config file).
2. Edge connects to `wss://api.ems.example/v1/edges/ws` (no query params).
3. Edge immediately sends authentication frame:
   ```json
   {
     "action": "authenticate",
     "edge_id": "site1_edge",
     "token": "sk_live_abc123..."
   }
   ```
4. Cloud validates API key against database; if invalid, closes connection.
5. Once authenticated, cloud may send commands to edge.
6. Edge receives commands, performs local validation, and writes to request channels.

### Rate Limiting Implementation

- **User command rate:** Redis key `rate_limit:user:{user_id}` with token bucket. Each command consumes one token; tokens refill at 10 per second.
- **Edge acknowledgment rate:** Redis key `rate_limit:edge:{edge_id}:acks` – higher limit (100/sec) to allow bursts. Tokens refill accordingly.
- **IP‑based limiting for HTTP endpoints:** Redis key `rate_limit:ip:{ip_address}` with sliding window counter.
- When limit exceeded, cloud sends appropriate error and logs.

### Replay Prevention

- Cloud maintains a Redis set `seen_commands` with TTL of 60 seconds.
- On command receipt:
  - Parse `timestamp`. If `abs(server_time - payload_timestamp) > 60s`, reject with `reason: "timestamp_too_old"`.
  - If within window, check if `command_id` exists in Redis set.
  - If present, reject with `reason: "duplicate_command"`.
  - If not, add `command_id` to set with TTL 60s and proceed.

### Session Revocation

- Django publishes to Redis channel `auth:revoked_users` when user permissions change.
- All cloud dispatcher instances subscribe to this channel. On receiving a user ID, they iterate through their active WebSocket connections and close those belonging to that user.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Cloud‑Side Dispatcher** | Enforces all security and rate limiting rules. |
| **Authentication Service (Django)** | Issues JWT for users; stores edge API keys (hashed); publishes revocation events. |
| **Redis** | Stores rate limit counters, seen command IDs, and pub/sub for revocation. |
| **Edge‑Side Dispatcher** | Validates incoming commands locally (second layer). |
| **Device Profiles** | Used for input validation (already covered in Command Validation). |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Authentication latency** | < 50 ms (p99) | Time to validate JWT/API key |
| **Per‑command JWT expiry check** | < 5 ms (p99) | Overhead of checking token expiry |
| **Rate limiting overhead** | < 10 ms (p99) | Time to check and increment counters |
| **Replay detection accuracy** | 100% of duplicates rejected | Test with same command_id within window |
| **Session revocation latency** | < 1 second from publish to close | Measure time from Django publish to WebSocket close |
| **Rate limit enforcement** | 100% of requests beyond limit rejected | Load test |
| **Security incident logs** | All security events logged | Audit log review |

---

## Implementation Notes

- **JWT Validation:** Use `python-jose` with caching of Django's public key (refresh every hour). Store the token's `exp` claim with the connection object (e.g., in a dictionary keyed by connection ID). On every command, check if `current_time > exp`. If expired, close connection with close code 4001.
- **Session Revocation:** Use Redis pub/sub. Each cloud dispatcher instance subscribes to `auth:revoked_users`. When a user ID is received, iterate over all active connections (maintained in a thread-safe map) and close those belonging to that user.
- **API Key Storage:** Store hashed API keys in database using bcrypt or Argon2. Never log full keys.
- **Rate Limiting:** Use Redis with Lua scripts for atomic token bucket operations to avoid race conditions.
- **Replay Cache:** Use Redis `SETEX` with 60‑second expiry. Ensure timestamp check is performed **before** cache lookup to avoid storing obviously stale commands.
- **CORS:** Configure in FastAPI using `fastapi.middleware.cors.CORSMiddleware` with explicit `allow_origins` list.
- **Input Sanitization:** Use Pydantic models for all incoming JSON; reject extra fields with `forbid="extra"`.
- **Logging:** Use structured logging with fields: `event_type` (auth_success, auth_failure, rate_limit, command, session_revoked), `user_id`, `edge_id`, `ip`, `command_id`. Ship to central logging system.

---

## Summary

The Security and Rate Limiting layer ensures that the EMS remains secure, reliable, and fair under all conditions. By implementing:

- **Strong authentication** with per‑command JWT expiry checks
- **Active session revocation** for immediate permission changes
- **Multi‑layer rate limiting** distinguishing user commands from edge acknowledgments
- **Robust replay prevention** combining timestamp validation with a command ID cache
- **Input validation and sanitization**
- **Comprehensive audit logging**

we protect the system from a wide range of threats while maintaining good performance and user experience.