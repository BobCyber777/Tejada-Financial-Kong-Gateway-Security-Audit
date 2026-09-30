# Tejada-Financial-Kong-Gateway-Security-Audit
Security reconnaissance, configuration review, remediation, and regression testing of the Kong Gateway used in the Tejada Financial Django fintech development environment.

# Tejada Financial — Kong Gateway Security Audit

Security reconnaissance, configuration review, remediation, and regression testing of the Kong Gateway used in the Tejada Financial Django fintech development environment.

## Scope

This assessment focused on the local gateway layer:

```text
Kali Security Workstation
        │
        │ HTTP
        ▼
Kong Gateway
        │
        │ HTTP upstream
        ▼
Django / Tejada Financial
```

### Components assessed

| Component                  |            Before |             After |
| -------------------------- | ----------------: | ----------------: |
| Kong Gateway               |             3.6.1 |             3.9.3 |
| Deployment mode            |           DB-less |           DB-less |
| Declarative configuration  |  `/kong/kong.yml` |  `/kong/kong.yml` |
| Proxy                      |          TCP/8000 |          TCP/8000 |
| Admin API host publication |          TCP/8001 |           Removed |
| Django upstream            | `172.18.0.1:9001` | `172.18.0.1:9001` |

## Assessment Method

The assessment followed a controlled sequence:

1. Network reconnaissance
2. Kong service and route enumeration
3. Upstream connectivity verification
4. Admin API exposure verification
5. Kong version upgrade
6. Configuration validation
7. Proxy regression testing
8. Admin API remediation
9. Post-remediation verification

No destructive testing was performed.

No unauthenticated administrative write operation was attempted.

---

# Findings

## KONG-01 — Kong Admin API Network Exposure

**Status: REMEDIATED**

### Initial condition

Kong Admin API TCP/8001 was published on all host interfaces:

```text
0.0.0.0:8001
[::]:8001
```

The Admin API was reachable remotely from the Kali security workstation without authentication.

Example:

```text
GET http://10.10.10.20:8001/status

HTTP/1.1 200 OK
Server: kong/3.6.1
```

The endpoint exposed operational information including:

* Kong runtime state
* worker information
* connection statistics
* Lua memory information
* configuration hash
* Kong configuration metadata

### Remediation

The host publication was removed from Docker Compose.

Before:

```yaml
ports:
  - "8000:8000"
  - "8001:8001"
```

After:

```yaml
ports:
  - "8000:8000"
```

The internal Kong Admin listener remains:

```yaml
KONG_ADMIN_LISTEN: 0.0.0.0:8001
```

but TCP/8001 is no longer published by Docker onto the host.

### Verification

Host listeners after remediation:

```text
0.0.0.0:8000
[::]:8000
```

No host TCP/8001 listener remained.

Remote test from Kali:

```text
curl http://10.10.10.20:8001/status
```

Result:

```text
Connection timed out
```

### Result

**Remote Admin API exposure closed.**

---

# KONG-02 — Gateway `/api` Path Configuration Mismatch

**Status: CONFIRMED — NOT YET REMEDIATED**

Kong is configured with:

```yaml
routes:
  - name: django-route
    paths:
      - /api
    strip_path: false
```

Django exposes:

```text
/health/
```

but not:

```text
/api/health/
```

Therefore:

```text
GET /api/health/
```

is forwarded by Kong to Django unchanged and produces:

```text
HTTP 404
```

The direct upstream health endpoint remains:

```text
GET http://172.18.0.1:9001/health/

HTTP/1.1 200 OK
{"status": "ok", "service": "tejada-financial"}
```

This is classified as a **gateway/application integration configuration defect**, not evidence of an upstream connectivity failure.

The configuration was intentionally left unchanged during the security remediation to maintain clean before/after evidence.

---

# KONG-03 — Kong → Django Upstream Connectivity

**Status: VERIFIED**

The Kong container was confirmed able to reach:

```text
172.18.0.1:9001
```

Direct upstream health check:

```text
HTTP/1.1 200 OK

{"status": "ok", "service": "tejada-financial"}
```

Kong proxy requests also reached Django successfully.

Therefore the previous gateway behavior was not caused by a persistent Kong-to-Django connectivity failure.

---

# KONG-04 — Kong Security Controls / Plugins

**Status: OBSERVATION**

Kong's bundled plugin implementations were available, but enumeration showed no configured plugins at:

```text
/plugins
/services/django-api/plugins
/routes/django-route/plugins
```

The assessment therefore did **not** treat bundled plugin availability as active security enforcement.

No evidence was found that authentication, rate limiting, ACL, or similar Kong plugins were configured on the assessed route.

This should be evaluated separately according to the intended production architecture.

---

# KONG-05 — Django DEBUG Information Disclosure

**Status: OPEN — SEPARATE APPLICATION FINDING**

During gateway testing, Django returned a detailed debug 404 page containing:

* Django URL patterns
* filesystem information
* request information
* framework/server details
* explicit `DEBUG = True` disclosure

Example:

```text
You're seeing this error because you have DEBUG = True
in your Django settings file.
```

This was observed in the development/lab environment.

It should be evaluated separately before any production deployment.

---

# Kong Upgrade

The gateway was upgraded in a controlled manner:

```text
Kong 3.6.1
    ↓
Kong 3.9.3
```

The image was pinned to:

```text
kong:3.9
```

The resolved image digest was:

```text
sha256:12972ce1ab6396083e56e7d46fce084836c98cc819344bef44a1f583ec3ab191
```

Post-upgrade:

```text
Kong version: 3.9.3
Status: healthy
```

Kong successfully loaded:

```text
/kong/kong.yml
```

No declarative configuration loading error was observed.

---

# Regression Results

## Kong health

```text
kong-gateway
IMAGE: kong:3.9
STATUS: Up (healthy)
VERSION: 3.9.3
```

## Direct Django health

```text
GET /health/

HTTP/1.1 200 OK

{"status": "ok", "service": "tejada-financial"}
```

## Kong proxy

```text
GET /api/health/

HTTP/1.1 404 Not Found
Via: 1.1 kong/3.9.3
```

The 404 is expected from the existing `/api` path configuration.

## Admin API after remediation

```text
GET http://10.10.10.20:8001/status

Connection timed out
```

This demonstrates that the host no longer exposes TCP/8001.

---

# Before / After

```text
BEFORE
======

Kali
 │
 ├── :8000 ──────► Kong Proxy ──────► Django
 │
 └── :8001 ──────► Kong Admin API
                    ▲
                    │
             UNAUTHENTICATED
             NETWORK EXPOSURE


AFTER
=====

Kali
 │
 └── :8000 ──────► Kong 3.9.3 ──────► Django
                         │
                         └── Admin API
                             internal container access
                             only

Host :8001
    CLOSED
```

---

# Evidence Principles

The assessment distinguishes between:

* confirmed vulnerabilities
* configuration defects
* environmental observations
* controls that were not tested
* assumptions that require further validation

No claim of unauthenticated administrative modification is made because no write operation against the Kong Admin API was attempted.

---

# Current Security Status

| Area                                   | Status                       |
| -------------------------------------- | ---------------------------- |
| Kong upgrade                           | Verified                     |
| Kong 3.9.3 healthy                     | Verified                     |
| Declarative configuration              | Verified                     |
| Kong → Django connectivity             | Verified                     |
| Admin API external exposure            | **Remediated**               |
| `/api` path mismatch                   | Open                         |
| Django DEBUG disclosure                | Open / environment dependent |
| Kong authentication/rate-limit plugins | Not configured               |
| Unauthenticated Admin API write access | Not tested                   |

---

## Environment

This repository documents a controlled security assessment of a development/lab environment.

It is not a certification, penetration-test attestation, regulatory compliance assessment, or production security guarantee.

Sensitive credentials, API keys, webhook secrets, private configuration, database credentials, and customer information must not be committed to this repository.

---

## Audit Principle

The goal of this repository is reproducible evidence:

```text
Observe
  ↓
Verify
  ↓
Document
  ↓
Change one variable
  ↓
Regression test
  ↓
Verify remediation
```

The Kong Admin API remediation demonstrates this process without changing the application routing behavior being independently investigated.
