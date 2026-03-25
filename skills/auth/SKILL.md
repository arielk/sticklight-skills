---
name: auth
description: Authentication and authorization patterns and implementation guidelines. Use when implementing login, signup, session management, OAuth, JWT, role-based access control, or any identity and access management features.
---

# Authentication & Authorization

Apply secure authentication and authorization best practices when implementing identity and access management features.

## Authentication Fundamentals

- Never store passwords in plain text — always hash with a strong, slow algorithm (bcrypt, scrypt, or Argon2)
- Use a sufficient salt and work factor (bcrypt cost >= 10)
- Implement rate limiting on login endpoints to prevent brute-force attacks
- Lock or delay accounts after repeated failed login attempts
- Require strong passwords (minimum 8 characters, mix of character types) but avoid overly complex rules that hurt usability
- Support multi-factor authentication (MFA/2FA) wherever possible

## Session Management

- Generate cryptographically random session tokens
- Store sessions server-side or use signed, encrypted cookies
- Set appropriate cookie flags: `HttpOnly`, `Secure`, `SameSite=Strict` (or `Lax`)
- Implement session expiration and idle timeouts
- Invalidate sessions on logout, password change, and privilege escalation
- Rotate session IDs after authentication to prevent session fixation

## JWT Best Practices

- Use short-lived access tokens (5-15 minutes) paired with longer-lived refresh tokens
- Always validate the `alg`, `exp`, `iss`, and `aud` claims
- Never store sensitive data in the JWT payload (it's only base64-encoded, not encrypted)
- Use asymmetric signing (RS256, ES256) for distributed systems; symmetric (HS256) only when issuer and verifier are the same service
- Store refresh tokens securely (httpOnly cookie or encrypted storage, never localStorage)
- Implement token revocation for refresh tokens

## OAuth 2.0 / OpenID Connect

- Use Authorization Code flow with PKCE for web and mobile apps
- Never use the Implicit flow for new applications
- Validate the `state` parameter to prevent CSRF
- Validate the `nonce` in ID tokens to prevent replay attacks
- Store client secrets securely — never expose them in client-side code
- Request the minimum required scopes

## Authorization & Access Control

- Implement the principle of least privilege — grant only the permissions needed
- Use role-based access control (RBAC) or attribute-based access control (ABAC)
- Check authorization on every request, server-side — never rely on client-side checks
- Return `403 Forbidden` for unauthorized access, `401 Unauthorized` for unauthenticated requests
- Log all access control decisions for audit purposes
- Separate authentication ("who are you?") from authorization ("what can you do?")

## Security Headers

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Content-Security-Policy: default-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
```

## Examples

### Password Hashing (Node.js)

```javascript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

async function hashPassword(password) {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password, hash) {
  return bcrypt.compare(password, hash);
}
```

### JWT Middleware (Express)

```javascript
import jwt from 'jsonwebtoken';

function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Authentication required' });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET, {
      algorithms: ['RS256'],
      issuer: 'your-app',
      audience: 'your-api',
    });
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```

### Role-Based Authorization Middleware

```javascript
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) return res.status(401).json({ error: 'Authentication required' });
    if (!allowedRoles.includes(req.user.role)) return res.status(403).json({ error: 'Insufficient permissions' });
    next();
  };
}

// Usage
app.delete('/api/users/:id', authenticate, authorize('admin'), deleteUser);
```

### Secure Cookie Configuration

```javascript
res.cookie('session', sessionId, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict',
  maxAge: 1800000, // 30 minutes
  path: '/',
});
```
