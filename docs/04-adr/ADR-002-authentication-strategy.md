# ADR-002: Authentication Strategy

| | |
|---|---|
| **ADR ID** | ADR-002 |
| **Title** | Authentication Strategy |
| **Status** | Proposed |
| **Author** | Architecture Team |
| **Date** | 2026-07-17 |
| **Project** | VArrow AI Finance |
| **Company** | VArrow.tech |

---

## Context

VArrow AI Finance requires a secure authentication strategy for its Identity bounded context (see [System Architecture §1](../05-architecture/SYSTEM_ARCHITECTURE.md)). The platform serves a single company's internal users across two client types — a React web application and a Flutter mobile application — both communicating with a single NestJS backend (per [ADR-001](ADR-001-technology-stack.md)).

### Requirements Driving This Decision

The following requirements from the [BRD](../02-brd/BRD.md) and [Business Module Catalog §1–§3](../02-brd/BUSINESS_MODULE_CATALOG.md) directly constrain the authentication design:

| Source | Requirement |
|---|---|
| FR-001 | Authenticate registered users and deny access to unverified or inactive accounts |
| FR-002 | Manage user sessions with configurable timeout policies |
| FR-003 | Enforce role-based access control across all operations |
| FR-004 | Enforce separation of duties for financial approvals |
| FR-007 | Record all authentication events and access control changes |
| FR-008 | Notify on suspicious or failed sign-in attempts |
| NFR-05 | Authentication shall use appropriate identity verification mechanisms |
| NFR-06 | Sessions shall be managed securely with configurable timeouts |
| NFR-09 | Sensitive financial data shall be protected in transit and at rest |
| BR-01 | Only registered, active users may access the platform |
| BR-04 | Separation of duties: submitter may not approve the same document |
| Catalog §1 | Sessions must expire according to company policy |
| Catalog §1 | Access must be denied when identity cannot be verified |
| Catalog §1 | Inactive users must not be able to sign in |

### Open Questions Acknowledged

The following BRD open questions (see [BRD §15](../02-brd/BRD.md)) remain unresolved. This ADR makes decisions that can accommodate a range of answers to these questions through configuration rather than code changes:

| OQ | Question | How This ADR Handles It |
|---|---|---|
| OQ-17 | Session timeout policies (idle timeout, max session)? | Token lifetimes are configurable via Administration Settings |
| OQ-18 | Password or identity verification requirements beyond standard practice? | Password policy is configurable; the architecture supports adding MFA in the future without structural change |

### Deployment Context

- **Single company, internal users only** — no public registration, no OAuth "Login with Google/GitHub" flows, no multi-tenant identity isolation.
- **Two client types** — web (React SPA) and mobile (Flutter) — both stateless from the server's perspective.
- **Single NestJS backend** — all API requests route through one backend application.
- **AWS deployment** — the platform runs on AWS (per ADR-001).

---

## Decision: Stateless JWT Authentication with Refresh Token Rotation

### 1. Token Architecture

Adopt a **stateless JSON Web Token (JWT)** authentication model with short-lived access tokens and long-lived, rotatable refresh tokens.

| Token | Purpose | Lifetime | Storage |
|---|---|---|---|
| **Access Token** | Authorizes API requests; contains user identity and role claims | Short-lived (configurable, default 15 minutes) | Web: in-memory only (JavaScript variable). Mobile: secure memory |
| **Refresh Token** | Obtains a new access token without re-entering credentials | Longer-lived (configurable, default 7 days) | Web: HTTP-only, Secure, SameSite=Strict cookie. Mobile: platform secure storage (Keychain / Keystore) |

### Reasoning

- **Stateless access tokens** eliminate server-side session storage, reducing backend complexity and enabling horizontal scaling if the modular monolith is eventually deployed across multiple instances.
- **Short-lived access tokens** limit the window of exposure if a token is compromised.
- **Refresh token rotation** (issuing a new refresh token with each use and invalidating the old one) provides replay detection — if a stolen refresh token is used after the legitimate user has already rotated it, the system detects the anomaly and can invalidate the entire token family.
- **HTTP-only cookies for refresh tokens on web** prevent JavaScript access, mitigating XSS-based token theft.
- **In-memory access tokens on web** mean they are never persisted to localStorage or sessionStorage, reducing the XSS attack surface.
- **Platform secure storage on mobile** (iOS Keychain, Android Keystore) provides hardware-backed protection for refresh tokens.

### Trade-offs

- **Stateless tokens cannot be instantly revoked.** If a user is deactivated (per FR-001, Catalog §1), the access token remains valid until it expires. With a 15-minute default lifetime, the maximum window is acceptable for a single-company deployment. For immediate revocation, a lightweight token blocklist checked at the API gateway level is available as an opt-in mechanism (see §5 below).
- **Refresh tokens require server-side storage** (in PostgreSQL) for rotation tracking and family invalidation. This is the only stateful component of the authentication system.
- **In-memory access tokens on web** are lost on page refresh, requiring a silent refresh via the refresh token cookie. This adds a brief latency on page load but is a standard and well-understood pattern.

---

### 2. Password Authentication

Adopt **email/username + password** as the primary credential type for Version 1.

| Concern | Decision |
|---|---|
| **Hashing** | Passwords shall be hashed using bcrypt with a configurable cost factor (default 12 rounds) |
| **Password policy** | Minimum length and complexity rules shall be configurable via Administration Settings (default: minimum 8 characters, at least one uppercase, one lowercase, one digit) |
| **Brute-force protection** | Rate-limit failed sign-in attempts per account (configurable threshold, default: 5 attempts in 15 minutes triggers temporary lockout) |
| **Account lockout** | Temporary lockout duration is configurable (default: 15 minutes). Administrators can manually unlock accounts |
| **Password change** | Users can change their own password (requires current password). Administrators can force a password reset |
| **Password reset** | Self-service password reset via a secure, time-limited, single-use token delivered through the Notification context (email) |

### Reasoning

- Email/password is the simplest credential type that meets the documented requirements for a single-company internal platform with no external identity provider integration in Version 1.
- Bcrypt is a well-established, timing-attack-resistant hashing algorithm with configurable cost factor, suitable for a financial application.
- Rate limiting and lockout directly address FR-008 (notify on suspicious sign-in attempts) and protect against credential stuffing.

### Trade-offs

- No multi-factor authentication (MFA) in Version 1. The architecture is designed to add TOTP-based MFA (see §7, Future Extensibility) without structural changes, but it is not implemented initially.
- No social login or SSO — appropriate for an internal platform but would need to be added if the deployment model ever changes.

---

### 3. Token Claims and Authorization

Access tokens shall carry the following claims:

| Claim | Type | Purpose |
|---|---|---|
| `sub` | string | User ID (unique, immutable identifier) |
| `email` | string | User email (for display and audit reference) |
| `roles` | string[] | Assigned role identifiers |
| `iat` | number | Token issued-at timestamp |
| `exp` | number | Token expiration timestamp |
| `jti` | string | Unique token identifier (for blocklist checks if enabled) |

**Authorization flow:**

1. Every API request includes the access token (in the `Authorization: Bearer <token>` header).
2. The NestJS authentication guard verifies the token signature and expiration.
3. The NestJS authorization guard reads the `roles` claim and evaluates it against the endpoint's required permissions.
4. Permissions are resolved from roles using the Roles & Permissions module's role-to-permission mapping (per [Catalog §3](../02-brd/BUSINESS_MODULE_CATALOG.md)).
5. Separation of duties enforcement (FR-004, BR-04) is handled by the Workflow Engine at the business logic level — it compares the submitting user's identity against the approving user's identity, independent of the authentication mechanism.

### Reasoning

- Embedding roles in the token avoids a database lookup on every request, supporting the stateless model.
- Separation of duties is a business rule, not an authentication concern — it belongs in the Workflow Engine, not in the token.
- The `jti` claim enables optional blocklist checking without requiring it on every request.

### Trade-offs

- If a user's roles change, the old roles remain in the access token until it expires (maximum 15 minutes). This is acceptable for a single-company deployment where role changes are infrequent and administratively controlled.
- The role claim increases token size. With the expected number of roles (< 10 per user), this adds negligible overhead.

---

### 4. Session Lifecycle

| Event | Behavior |
|---|---|
| **Sign in** | Validates credentials → issues access token + refresh token → records `UserAuthenticated` event (per System Architecture §1) |
| **Sign-in failure** | Records `AuthenticationFailed` event → increments rate-limit counter → triggers notification if threshold reached (FR-008) |
| **Inactive user sign-in** | Denies access immediately, regardless of credential validity (Catalog §1) |
| **Token refresh** | Validates refresh token → rotates (issues new pair, invalidates old refresh token) → returns new access token |
| **Sign out** | Invalidates the user's current refresh token family → clears refresh token cookie (web) / removes from secure storage (mobile) |
| **Idle timeout** | Access token expires naturally. If idle beyond refresh token lifetime, the user must re-authenticate |
| **Max session duration** | Refresh token has an absolute maximum lifetime (configurable). After expiry, the user must re-authenticate regardless of activity |
| **User deactivation** | Invalidates all refresh tokens for the user. Active access tokens expire naturally (< 15 minutes max) |
| **Password change** | Invalidates all refresh tokens across all devices for the user |
| **Forced sign-out (admin)** | Invalidates all refresh tokens for the target user |

### Reasoning

- Aligns with FR-002 (configurable timeout policies), NFR-06 (secure session management), and Catalog §1 (sessions expire per policy).
- Refresh token family invalidation on sign-out, password change, and deactivation ensures that compromised tokens from other devices are revoked.
- Natural access token expiration for deactivation (rather than real-time revocation) is a deliberate trade-off for single-company deployment simplicity.

---

### 5. Token Revocation Strategy

For Version 1, **natural expiration is the primary revocation mechanism** — access tokens expire in 15 minutes, and refresh tokens are rotated and invalidated on sign-out, password change, and deactivation.

**Optional immediate revocation:** A lightweight blocklist (backed by an in-memory cache with PostgreSQL persistence) can be enabled via Administration Settings for organizations that require sub-minute revocation. When enabled:

1. The `jti` from revoked access tokens is added to the blocklist.
2. The authentication guard checks incoming tokens against the blocklist.
3. Blocklist entries are automatically purged after the token's `exp` time has passed.

### Reasoning

- For a single-company deployment with < 100 expected users, the 15-minute natural expiration window is an acceptable trade-off that avoids the complexity of real-time revocation infrastructure.
- The opt-in blocklist is available for high-security scenarios without imposing its overhead on every deployment.

### Trade-offs

- Without the blocklist enabled, a deactivated user's access token remains valid for up to 15 minutes.
- With the blocklist enabled, every API request incurs a cache lookup (negligible latency for in-memory cache, but adds operational complexity).

---

### 6. Cross-Platform Token Management

| Platform | Access Token | Refresh Token | Notes |
|---|---|---|---|
| **Web (React SPA)** | In-memory (JavaScript variable) | HTTP-only, Secure, SameSite=Strict cookie | Silent refresh on page load via `/auth/refresh` endpoint; cookie sent automatically by browser |
| **Mobile (Flutter)** | Secure in-memory | iOS Keychain / Android Keystore | Refresh on app launch or token expiration; secure storage APIs used for persistence across app restarts |

### Reasoning

- **Web:** HTTP-only cookies are inaccessible to JavaScript, eliminating the most common XSS token theft vector. SameSite=Strict prevents CSRF attacks involving the refresh endpoint. In-memory access tokens are never written to localStorage.
- **Mobile:** Platform-provided secure storage (Keychain/Keystore) offers hardware-backed encryption. This is the Flutter best practice for sensitive credential storage.

### Trade-offs

- Web: SameSite=Strict cookies may not be sent in cross-origin scenarios. Since the platform is a single-company internal deployment (same origin), this restriction is acceptable.
- Mobile: Secure storage APIs vary slightly between iOS and Android. Flutter packages (e.g., `flutter_secure_storage`) abstract this, but platform-specific behavior must be tested.

---

### 7. Future Extensibility

This ADR is designed to support the following future enhancements documented in the [Business Module Catalog §1](../02-brd/BUSINESS_MODULE_CATALOG.md) without requiring a new ADR or structural change:

| Future Capability | How This Architecture Supports It |
|---|---|
| **Multi-factor authentication (TOTP)** | Add a TOTP verification step after password validation. The token issuance flow gains an intermediate state ("credentials verified, MFA pending") before issuing the access token. No change to token format or storage. |
| **External identity provider / SSO** | The Identity context boundary (per System Architecture) isolates authentication from the rest of the platform. An OAuth 2.0 / OIDC integration replaces the password verification step; the token issuance and claims structure remain identical. |
| **Adaptive access policies** | The `roles` claim can be extended with contextual claims (IP, device, location) for risk-based access decisions. The authorization guard evaluates additional claims without changing the token architecture. |
| **Microservices extraction** | If the Identity context is extracted (per System Architecture §1, Future Scalability), the JWT-based stateless model requires no session synchronization between services — any service can validate the access token independently using the public key. |

---

### 8. Audit and Compliance

All authentication events are published as domain events (per [System Architecture §1, Events Published](../05-architecture/SYSTEM_ARCHITECTURE.md)):

| Event | When Published | Consumed By |
|---|---|---|
| `UserAuthenticated` | Successful sign-in | Administration (Audit Logs), Reporting |
| `AuthenticationFailed` | Failed sign-in attempt | Administration (Audit Logs), Notification |
| `UserDeactivated` | Account deactivated (triggers token invalidation) | Workflow Engine, Administration, Notification |

These events satisfy:
- FR-007 (record all authentication events)
- FR-008 (notify on suspicious sign-in attempts)
- NFR-10 (immutable audit trail for significant actions)
- Catalog §1 Business Rule: "Access must be denied when identity cannot be verified" — denial events are recorded.

---

## Consequences

### Positive

- **Stateless API layer** — no server-side session store for access tokens, reducing backend complexity and supporting future horizontal scaling.
- **Cross-platform consistency** — the same token format and authorization model works for both web and mobile, simplifying the backend API.
- **Configurable policies** — session timeouts, password rules, and lockout thresholds are all configurable via Administration Settings, directly addressing BRD open questions OQ-17 and OQ-18 without requiring code changes.
- **Security-by-default** — HTTP-only cookies, in-memory tokens, bcrypt hashing, rate limiting, and refresh rotation collectively provide defense-in-depth appropriate for a financial operations platform.
- **Future-proof** — the architecture supports MFA, SSO, and microservices extraction without structural changes.
- **Aligned with existing architecture** — Identity context events, Administration Settings integration, and Notification triggers all follow the patterns defined in the System Architecture.

### Negative

- **15-minute revocation window** — without the optional blocklist, deactivated users retain access for up to 15 minutes. Acceptable for single-company deployment, but must be communicated to stakeholders.
- **Refresh token server-side storage** — adds a stateful component (PostgreSQL table for refresh token families) that must be maintained and cleaned up.
- **Token refresh latency on web** — page refresh triggers a silent token refresh, adding a brief delay before the first authenticated API call.
- **No MFA in Version 1** — relies on password-only authentication initially. The risk is mitigated by the internal deployment model (no public internet exposure for authentication) and the option to add TOTP without structural changes.

---

## Alternatives Considered

### 1. Server-Side Sessions (Express Session / Cookie-Based)

**Description:** Traditional server-side session storage with a session ID cookie.

**Why rejected:**
- Requires session storage (Redis, database) that must be shared across instances if the application scales horizontally.
- Creates a stateful coupling between the web client and a specific server instance (or requires sticky sessions / shared session store).
- Does not translate cleanly to mobile clients (Flutter), which would need a different authentication mechanism.
- Adds operational complexity for a startup with a small team.

### 2. OAuth 2.0 Authorization Server (Self-Hosted)

**Description:** Deploy a full OAuth 2.0 authorization server (e.g., Keycloak, ORY Hydra) for identity management.

**Why rejected for Version 1:**
- Significant operational overhead — a separate service to deploy, maintain, upgrade, and monitor.
- Overengineered for a single-company internal platform with no external identity federation requirements in Version 1.
- The selected JWT approach achieves the same token-based stateless architecture without the infrastructure burden.
- The architecture supports migrating to an external identity provider in the future (see §7) without committing to it now.

### 3. Opaque Tokens with Introspection

**Description:** Issue opaque (non-JWT) access tokens that require server-side validation on every request via an introspection endpoint.

**Why rejected:**
- Requires a network call to the introspection endpoint for every API request, adding latency and a single point of failure.
- Negates the performance and simplicity benefits of stateless JWTs.
- For a single-company modular monolith, the added infrastructure cost is not justified.
- Immediate revocation (the primary advantage) is available via the optional JWT blocklist approach without requiring introspection.

### 4. Passwordless / Magic Link Authentication

**Description:** Eliminate passwords in favor of email-based magic links or passkeys.

**Why rejected for Version 1:**
- Requires reliable email delivery infrastructure from day one (the Notification context's email channel may not be the first channel implemented).
- Unfamiliar to many users in financial operations environments where password-based login is the established norm.
- Passkey support is still maturing across browsers and mobile platforms.
- Can be added as an alternative authentication method in a future version without changing the token architecture.

---

## Summary of Decisions

| Concern | Decision |
|---|---|
| Token type | JWT (stateless access tokens + rotatable refresh tokens) |
| Access token lifetime | Configurable, default 15 minutes |
| Refresh token lifetime | Configurable, default 7 days |
| Credential type (V1) | Email/username + password (bcrypt, configurable policy) |
| Web token storage | Access: in-memory. Refresh: HTTP-only Secure SameSite=Strict cookie |
| Mobile token storage | Access: secure memory. Refresh: platform secure storage (Keychain/Keystore) |
| Brute-force protection | Rate limiting + temporary account lockout (configurable thresholds) |
| Token revocation | Natural expiration (primary) + optional blocklist (opt-in for immediate revocation) |
| Audit | All auth events published as domain events per System Architecture §1 |
| MFA | Not in Version 1; architecture supports TOTP addition without structural changes |
| SSO / external IdP | Not in Version 1; architecture supports future integration via Identity context boundary |

---

## Governing References

| Document | Relevance |
|---|---|
| [BRD](../02-brd/BRD.md) §6.1 | Functional requirements FR-001 through FR-008 |
| [BRD](../02-brd/BRD.md) §7 | Non-functional requirements NFR-05, NFR-06, NFR-09, NFR-10 |
| [BRD](../02-brd/BRD.md) §8 | Business rules BR-01, BR-04 |
| [BRD](../02-brd/BRD.md) §15 | Open questions OQ-17, OQ-18 |
| [Business Module Catalog](../02-brd/BUSINESS_MODULE_CATALOG.md) §1–§3 | Authentication, User Management, Roles & Permissions module definitions |
| [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md) §1 | Identity bounded context — events, dependencies, and interactions |
| [ADR-001](ADR-001-technology-stack.md) | Technology stack — NestJS, React, Flutter, PostgreSQL, AWS |

---

*This ADR proposes the authentication strategy for VArrow AI Finance. It is derived entirely from the existing project documentation and makes no assumptions beyond what is documented. Future changes to this strategy must be captured in subsequent ADRs.*
