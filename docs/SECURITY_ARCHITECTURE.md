# Security Architecture

HOROOMART holds a business's cash positions, bank account numbers, salaries and customer
contact details. Its security model is defence in depth: each layer assumes the one in
front of it might fail.

## Authentication

| Control | Behaviour |
|---|---|
| **Password policy** | Configurable **per organization**: minimum length plus required character classes (mixed case, letters, numbers). Applied to password set, change and reset alike. |
| **Two-factor authentication** | TOTP (RFC 6238, any standard authenticator app). **Mandatory** for anyone holding system-administration permission; available to every other user. Secrets are encrypted at rest. |
| **TOTP replay protection** | A one-time code is accepted **once**. Re-submitting a code already used within its validity window is rejected, which defeats shoulder-surfed or intercepted codes. |
| **Trusted devices** | A user may mark a device as trusted to skip the second factor on it. Administrators can list and **block** trusted devices, revoking that trust immediately. |
| **Brute-force throttling** | Login, password-reset and verification endpoints are rate-limited per client. |
| **Forced password change** | When an administrator resets a password, the user must set their own before reaching anything else. |
| **Session invalidation** | Changing a password terminates the user's **other** sessions, so an attacker holding a stolen session loses it the moment the owner recovers the account. |
| **No account enumeration** | Password-reset responses are identical whether or not the email exists. |
| **Per-tenant session lifetime** | Each organization sets its own idle-session timeout. |

## Authorization

- **Role-based access control** with fine-grained permissions (Spatie Permission). Roles are
  **organization-scoped**: each tenant defines its own roles, and one tenant's role changes
  can't affect another tenant.
- **Tenant isolation** runs as middleware on every authenticated request and as query scopes
  on operational data. See [ARCHITECTURE.md](ARCHITECTURE.md#multi-tenancy).
- **Segregation of duties** is enforced in code: nobody approves a record they created. See
  [FINANCIAL_INTEGRITY.md](FINANCIAL_INTEGRITY.md#approval-and-segregation-of-duties).
- **API access** uses revocable per-user tokens. The API applies the same permission checks as
  the web interface, with a regression test guarding against an endpoint being exposed
  without them.

## Data protection

**Field-level encryption** (application-level, AES-256 via the framework's encrypter) covers
these fields:

| Record | Encrypted fields |
|---|---|
| Bank accounts | account number, SWIFT code |
| Employee and user records | base salary |
| Customers | phone number |
| Users | two-factor secret |

For these fields, a copy of the database alone is not enough to read the values; the
application key, held outside the database, is also required.

Other personal and operational data is **not** encrypted at the field level. That includes
contact details other than the fields above, and the pay amounts recorded on individual
payroll lines. It is protected by authentication, role-based permissions, tenant isolation
and restricted database access. Extending field-level encryption to payroll line amounts is
planned.

**Audit logs redact sensitive fields.** Passwords, remember tokens, two-factor secrets and
salary values are stripped from the before/after snapshots written to the audit trail, so
the audit trail never becomes a second copy of the secrets it exists to protect.

**Backups** run daily with 14-day retention, are replicated off-site, and are verified daily.
Verification failures alert administrators. Database restores are a controlled,
permission-gated and audited operation, not an ad-hoc shell task.

## Application hardening

| Threat | Control |
|---|---|
| **Cross-site scripting** | Output escaping by default; a **strict Content Security Policy** with no `unsafe-eval`, made possible by using Alpine.js's CSP-compatible build; regression tests that store script payloads in user-editable fields and assert they render inert |
| **Clickjacking** | `X-Frame-Options` and CSP frame restrictions |
| **Protocol downgrade** | HTTP Strict Transport Security |
| **MIME sniffing** | `X-Content-Type-Options: nosniff` |
| **Referrer leakage** | Restrictive `Referrer-Policy` |
| **Unneeded browser features** | `Permissions-Policy` disables camera, microphone and geolocation |
| **CSRF** | Framework CSRF tokens on every state-changing web request |
| **SQL injection** | Parameterized queries throughout (query builder / ORM) |
| **Price tampering** | Prices are resolved server-side; client-supplied prices are ignored |
| **Mass assignment** | Explicit fillable whitelists; no model is left unguarded |
| **Resource exhaustion via reports** | Report date ranges are capped, and heavy exports run on a background queue rather than in the request |

## Verification

Security properties are covered by automated tests that run with every change, among them:
two-factor setup and replay rejection, session invalidation on password change, reset
enumeration resistance, CSP header content, stored-XSS inertness, middleware ordering, API
guard coverage and segregation-of-duties enforcement. The system has also been through
multiple independent security and production-readiness reviews, with every finding tracked to
closure.

## Reporting a vulnerability

If you believe you have found a security issue in HOROOMART, please contact HOROO Innovations
privately through the organization's GitHub profile rather than opening a public issue.
