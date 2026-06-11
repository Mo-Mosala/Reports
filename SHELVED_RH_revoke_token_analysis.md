# SHELVED — RH OAuth revoke_token 500 / session-revocation analysis

**Status:** Not submitted. Confirmed Low/Informational; session-persistence hypothesis disproven.
**Asset:** `api.robinhood.com`
**Last verified:** 10 June 2026
**Test account:** mora.mosala@gmail.com (user_id `3b53d08e-ec22-4c01-bc19-5f0b540dbf0b`)

---

## Why shelved (read first)

The interesting hypothesis — *"access token survives logout → compromised-session persistence"* — was tested rigorously and is **false**. Logout revokes the access token within ~60 seconds and it stays revoked. What remains (a 500 on the dedicated revoke endpoint) is a real functional/standards bug with **no demonstrated security impact**, because normal logout-based revocation works. Not worth signal. Banked here so the surface is never re-walked.

---

## Confirmed facts

**1. `POST /oauth2/revoke_token/` returns HTTP 500 for all inputs.**
Reproduced 10 Jun 2026: 15 consecutive requests, all 500, each a unique `trace-uuid` (independent crashes, not cached). All three backend tracks affected: `sheriff-server`, `sheriff-server-canary`, `sheriff-server-baseline` (`*.sheriff.svc.cluster.local:80`). RFC 7009 §2.2 requires 200 even for invalid tokens, so this endpoint performs no revocation.

**2. `x-envoy-decorator-operation` header leaks internal k8s topology** on those 500s to unauthenticated callers (service name, namespace `sheriff`, cluster domain, port, deployment track). `x-robinhood-api-version: 0.0.0` also disclosed. Informational alone — no access granted without a separate primitive (SSRF/smuggling), none demonstrated.

**3. Logout-based revocation WORKS (this is what kills the report).**
Captured live access token, confirmed `BEFORE: 200`, logged out of all devices, replayed token-only (`credentials:'omit'`):

| Check | Result |
|---|---|
| t+0 | 200 |
| t+1min | 401 |
| t+5min | 401 |
| t+30min | 401 |

The ~60s window at t+0 is normal distributed-revocation propagation lag, not an exploitable condition. The broken `/revoke_token/` endpoint is therefore bypassed entirely by the working logout path.

---

## Conclusion for negatives.md

> `api.robinhood.com` OAuth session revocation: access token is revoked on "log out all devices" within ~60s propagation and stays revoked (verified to t+30m, token-only/credentials:omit). `/oauth2/revoke_token/` returns 500 across all tracks (RFC 7009 violation) but has no security impact since logout revocation works. Envoy topology header (`x-envoy-decorator-operation`) leaks internal service names on 500s — info only. **No submittable finding. Do not re-test without a new SSRF/smuggling primitive that would activate the topology disclosure.**

---

## If ever reactivated
The topology disclosure (#2) only becomes meaningful if a separate finding gives external→internal reachability (SSRF, request smuggling, internal-route confusion). Park condition: if such a primitive is found against any in-scope host, revisit — the leaked `sheriff` service/namespace/port becomes a target map.
