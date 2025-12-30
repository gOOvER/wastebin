# Security Improvements

This document covers the security enhancements made during a comprehensive security review.

## Critical Fix: Hardcoded Salt Removed

The application shipped with a hardcoded default salt `"somesalt"`, making all installations vulnerable to rainbow table attacks. This has been fixed.

**What changed:**
- The `WASTEBIN_PASSWORD_SALT` environment variable is now mandatory
- Application won't start without it
- Error message includes instructions: `openssl rand -base64 32`

*Modified: `crates/wastebin_core/src/env.rs`*

## Other Security Enhancements

### Password Strength Validation

Empty or weak passwords were previously accepted. Now enforcing a minimum of 8 characters.

- Added `WeakPassword` error variant with clear messaging
- Validation happens before encryption
- *Modified: `crates/wastebin_core/src/db.rs`*

### CSRF Protection for Deletions

Delete operations used GET requests, making them vulnerable to CSRF attacks.

Fixed by changing the route to POST and using a proper form submission instead of a simple link.

*Modified: `crates/wastebin_server/src/main.rs`, `crates/wastebin_server/templates/paste.html`*

### User Enumeration Prevention

The `delete_for()` function previously revealed whether an entry existed based on different error messages.

Simplified the implementation to return a consistent error regardless of whether the entry doesn't exist or the UID doesn't match.

*Modified: `crates/wastebin_core/src/db.rs`*

### Rate Limiting

**Note:** Rate limiting using `tower-governor` was attempted but removed due to layer compatibility issues with Axum 0.8. This should be reconsidered when upgrading dependencies or using an alternative rate limiting solution. Consider implementing rate limiting at the reverse proxy level (nginx, Caddy) for now.

## Required Configuration

Before starting the application, you must set:

```bash
export WASTEBIN_PASSWORD_SALT=$(openssl rand -base64 32)
```

Or for persistence:

```bash
echo "WASTEBIN_PASSWORD_SALT=$(openssl rand -base64 32)" >> .env
```

**Important:** Use a different salt for each installation.

## Not Implemented (Low Priority)

### Rate Limiting
Attempted with `tower-governor` but had compatibility issues. Recommend implementing at reverse proxy level (nginx/Caddy) or revisiting when dependencies are updated.

### Decompression Bomb Protection
Consider checking decompressed size before full extraction and setting limits for uncompressed data.

### Security Event Logging
Would be useful to log failed authentication attempts, deletion attempts with wrong UIDs, and implement a proper audit trail.

### Timing Attack Mitigation
While Argon2 already provides some protection, additional constant-time comparison could be considered for password verification.

## Testing the Changes

Set the environment variable for tests:
```bash
# PowerShell
$env:WASTEBIN_PASSWORD_SALT="test_salt_for_unit_tests"
cargo test

# Bash/Linux
WASTEBIN_PASSWORD_SALT="test_salt_for_unit_tests" cargo test
```

Manual testing:
- Password validation: Try creating a paste with password <8 chars, should fail
- CSRF protection: Try DELETE via GET request, should not work (405 Method Not Allowed)
- User enumeration: Try deleting with wrong UID, same error as non-existent entry

## Migration Notes

For existing installations:

1. Generate and set `PASSWORD_SALT` before updating
2. Users will need to use passwords with at least 8 characters
3. Test rate limiting against your expected traffic patterns
4. Adjust limits in `make_app()` if needed

## Security Checklist

What's now in place:
- [x] No hardcoded default salt
- [x] Password strength enforcement (min. 8 characters)
- [x] CSRF protection on state-changing operations (POST instead of GET)
- [x] User enumeration prevented
- [ ] Rate limiting (recommend implementing at reverse proxy level)
- [x] SQL injection protection (parameterized queries)
- [x] XSS protection (template auto-escaping)
- [x] Security headers configured
- [x] Secure cookie handling
- [x] Modern cryptography (XChaCha20-Poly1305, Argon2i)

## Test Results

All 16 tests passing:
```
test result: ok. 16 passed; 0 failed; 0 ignored
```
