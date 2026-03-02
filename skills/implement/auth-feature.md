# Adding Auth Features to SMG

Two-factor auth: API key (SHA-256) + JWT/OIDC. Roles: Admin (control plane) and User (data plane).

## Steps

### Adding a New Role

1. Extend `Role` enum in `auth/src/`
2. Update permission checks in middleware
3. Update role mapping for JWT claims
4. Add audit logging for new role actions

### Adding a New OIDC Provider

1. Configure `JwtConfig` with provider's JWKS URI
2. Set `role_claim` to match provider's claim name (default: `"role"`)
3. Map IDP roles to gateway roles via `role_mapping`:
   ```rust
   role_mapping: HashMap::from([
       ("admin".to_string(), Role::Admin),
       ("reader".to_string(), Role::User),
   ])
   ```
4. Test: Token validation, claim extraction, role mapping

### Adding a Custom Auth Method

1. Implement validation logic in `auth/src/`
2. Extract `Principal` from request
3. Integrate in middleware chain (`/admin/*` routes)
4. Add audit event for the new method

### JWT Flow (7 Steps)

1. Decode header → extract `kid`
2. Fetch signing key from JWKS (cached, TTL 1hr)
3. Verify algorithm matches key
4. Validate: expiry, issuer, audience
5. Extract role from `role_claim`
6. Map via `role_mapping`
7. Check JTI cache for replay (10k token cache)

## Audit Events

All control plane mutations must be logged:
```rust
AuditEvent { timestamp, request_id, principal_id, operation, resource, outcome }
```

## Key Rules

- API keys: SHA-256 hash at load time, constant-time comparison
- JWT: Always validate expiry, issuer, audience
- Audit: Log all admin operations (add/remove modules, config changes)
- Never store plaintext keys
