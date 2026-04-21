# High-Security Addon
### Forge Addon | Append for enterprise, government, healthcare, or finance projects

---

## WHEN TO USE

- Government applications
- Healthcare (HIPAA)
- Finance (PCI DSS, SOX)
- Legal tech
- Enterprise with strict compliance requirements
- Any app handling highly sensitive personal data

---

## BASELINE (BEYOND UNIVERSAL RULES)

### Authentication Hardening
- **Multi-factor authentication (MFA) required** for all users, not optional
- Support TOTP (Google Authenticator, Authy) at minimum
- Consider WebAuthn/FIDO2 for passwordless
- **Account lockout** after 5 failed login attempts (15-minute cooldown)
- **Password policy**: 12+ characters minimum, mixed case, numbers, symbols
- Check passwords against HaveIBeenPwned (pwnedpasswords.com) API on signup and change
- Force password rotation every 90 days for privileged accounts (debatable — NIST now recommends against forced rotation unless compromise suspected)
- **Session timeout**: 15 minutes idle, 12 hours absolute maximum
- Revoke all sessions on password change

### Session Management
- Use secure, HTTP-only, SameSite=Strict cookies
- Bind sessions to IP address or device fingerprint (with graceful handling for mobile users)
- Store session state server-side, not just in JWT
- Detect and alert on concurrent logins from different locations

---

## DATA ENCRYPTION

### At Rest
- Database encryption enabled (TDE on SQL Server, or disk-level encryption)
- Encrypt sensitive columns beyond database-level (field-level encryption for PII, SSNs, etc.)
- Use a proper KMS (AWS KMS, Azure Key Vault, HashiCorp Vault) — never DIY key management

### In Transit
- TLS 1.3 minimum (TLS 1.2 acceptable, older versions forbidden)
- HSTS header with `max-age=31536000; includeSubDomains; preload`
- Certificate pinning for mobile apps
- No self-signed certificates in production

### Backups
- Encrypted backups
- Tested restore procedures (not just backup existence)
- Off-site storage
- Retention policy matching compliance requirements

---

## AUDIT LOGGING

### What to Log (Every Event)
- Authentication attempts (success and failure)
- Permission/role changes
- Access to sensitive data
- Data modifications (who, what, when, before/after values)
- Exports and downloads
- Administrative actions
- Configuration changes

### Log Properties
- Tamper-evident (hash-chained or append-only storage)
- Centralized (SIEM or centralized log platform)
- Retained per compliance requirement (7 years typical for finance, varies for healthcare)
- Accessible for audit without exposing to attackers

### What NOT to Log
- Passwords (even hashed)
- Full credit card numbers
- Social security numbers
- Full PII unless explicitly required

---

## ACCESS CONTROL

### Principle of Least Privilege
- Every user gets the minimum access needed
- Every service account gets the minimum API access needed
- Review and revoke quarterly

### Role-Based Access Control (RBAC)
- Define roles with specific permissions
- Assign users to roles, not individual permissions
- Changes to roles are audit events

### Attribute-Based Access Control (ABAC) — When Needed
- For complex permission logic (department, clearance level, time of day)
- More flexible than RBAC but harder to reason about
- Document policies clearly

### Separation of Duties
- No single person can execute sensitive operations end-to-end
- Example: One person creates a payment, another approves it
- Break-glass procedures for emergencies (with heavy logging)

---

## SECRETS MANAGEMENT

### Never in Code
- Never in source files
- Never in environment variable files committed to git
- Never in Docker images

### Where Secrets Live
- **Development**: Local `.env` file (in `.gitignore`)
- **Staging/Production**: Secrets manager (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, Vault)
- Rotate secrets regularly
- Audit secret access

### Application Secrets
- Database passwords
- API keys for third-party services
- JWT signing keys
- Encryption keys

---

## INPUT VALIDATION — EXTRA STRICT

### Beyond Standard Validation
- Content-type validation (reject unexpected types)
- Max request size limits
- Max file upload sizes with mime-type verification (check magic bytes, not just extension)
- Reject null bytes
- Reject overly long strings even if type is correct

### SQL Injection — Zero Tolerance
- ORMs only, no raw SQL in business logic
- If raw SQL is absolutely required: parametrized queries always, never string concatenation
- Regular SAST (static analysis) scans

### Command Injection
- Never pass user input to shell commands
- Never use `eval`, `exec`, `system` with user input
- If shell commands unavoidable, use subprocess arrays (not strings) and allowlists

### XSS
- Escape all output in templates
- Use frameworks that auto-escape (React, Vue do this by default)
- Strict Content-Security-Policy (no `unsafe-inline`, no `unsafe-eval`)

---

## HEADERS (EVERY RESPONSE)

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'
```

Adjust CSP based on specific needs — tighter is better.

---

## RATE LIMITING — GRANULAR

### Different Limits for Different Endpoints
- Authentication: 5 attempts per 15 minutes per IP AND per account
- Public APIs: 60 requests per minute per IP
- Authenticated APIs: Higher limits per user
- Sensitive operations (password reset, data export): Very strict limits

### Behavior Under Limit
- 429 status code
- `Retry-After` header
- Don't leak information about limits in error messages

---

## VULNERABILITY MANAGEMENT

### Dependency Scanning
- Automated dependency scanning on every PR
- Tools: Dependabot, Snyk, npm audit, pip-audit, `dotnet list package --vulnerable`
- Patch critical vulnerabilities within 24 hours
- Patch high vulnerabilities within 1 week

### SAST (Static Application Security Testing)
- Automated on every PR
- Tools: Semgrep, CodeQL, Snyk Code
- Fix findings before merge

### DAST (Dynamic Application Security Testing)
- Run against staging before production releases
- Tools: OWASP ZAP, Burp Suite

### Penetration Testing
- Annual third-party pentest minimum
- After major feature releases
- Scope documented clearly

---

## COMPLIANCE CONSIDERATIONS

### HIPAA (US Healthcare)
- Business Associate Agreement (BAA) required with cloud providers
- PHI (Protected Health Information) encrypted at rest and in transit
- Audit logs with 6-year retention
- Access controls with automatic logoff
- Breach notification procedures

### PCI DSS (Payments)
- See payments.md addon
- Never store card data
- Use tokenization via payment provider
- Network segmentation if storing any cardholder data

### GDPR (EU)
- Lawful basis for every data collection
- User consent tracking
- Right to access, delete, export (data portability)
- Data Processing Agreements with third parties
- Breach notification within 72 hours

### SOC 2 (Enterprise B2B)
- Formal policies and procedures
- Access reviews
- Change management
- Incident response plan
- Annual audit

---

## INCIDENT RESPONSE

### Preparation
- Documented incident response plan
- Team roles and contact information
- Runbooks for common scenarios
- Communication templates

### Detection
- Monitoring and alerting
- Anomaly detection
- User-reported issues triage

### Response
- Contain (isolate affected systems)
- Investigate (preserve evidence)
- Eradicate (fix root cause)
- Recover (restore service)
- Lessons learned (post-mortem)

### Notification
- Internal stakeholders immediately
- Customers per contract/regulation
- Regulators per legal requirements

---

## DEVELOPMENT PRACTICES

### Code Review
- Required for all changes
- At least one reviewer
- Security-focused review for sensitive changes
- Automated checks must pass

### Branch Protection
- Main branch requires PR
- No force-pushes to main
- Required status checks (CI, tests, security scans)
- Signed commits for high-security repositories

### Supply Chain Security
- Pin all dependencies
- Verify checksums where possible
- Use private package registries for internal libraries
- Sign and verify container images

---

## OBSERVABILITY FOR SECURITY

### Alert On
- Repeated failed authentication attempts
- Privilege escalations
- Access from unusual locations
- Mass data access patterns
- Configuration changes
- Spike in error rates (could indicate attack)

### Don't Alert On
- Every login (noise)
- Every API call (noise)
- Routine operations

---

## USER EDUCATION

- Security policy documentation
- Anti-phishing training (for internal users)
- Clear communication about what the app does and doesn't do
- Privacy policy written in plain language
- Secure-by-default settings

---

## SPECIFIC DO NOTS

- Never roll your own crypto
- Never use MD5, SHA1 for anything security-related
- Never use predictable random (use `secrets` module in Python, `crypto.randomBytes` in Node)
- Never log full PII or secrets
- Never rely on client-side validation alone
- Never trust user input — ever
- Never store passwords reversibly encrypted (hashing only)
- Never use HTTP in production (HTTPS only, no redirects from HTTPS to HTTP)
- Never commit secrets, even to private repos
- Never skip security headers "temporarily"
- Never grant blanket admin access
- Never disable security features for "ease of development" in production

---

## SESSION COMPLETION CHECKLIST — HIGH-SECURITY SPECIFIC

- [ ] MFA implemented and enforced
- [ ] All secrets in secrets manager (not env files in production)
- [ ] Security headers configured
- [ ] Audit logging in place for sensitive actions
- [ ] Rate limiting on all endpoints
- [ ] Dependency scan clean
- [ ] SAST scan clean
- [ ] Input validation strict
- [ ] Encryption at rest and in transit verified
- [ ] Backup encryption verified
- [ ] Session management secure
- [ ] Documented security architecture

---

*This addon is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
