# Sentry JWT and OIDC Support for Workload Identity

* Author(s): Joni Collige
* State: Implemented
* Updated: 2025-05-24

## Overview

This proposal adds support for configuring Dapr Sentry to issue JWTs as an additional credential alongside the existing x.509 certificate. The JWT will be signed with a configurable key and algorithm, and a JWKS endpoint will be provided for trust distribution. Optionally, Sentry can host a partial OIDC discovery server to allow third-party identity providers to federate trust using the `.well-known/openid-configuration` and `jwks.json` endpoints. The JWT will be returned in the existing `SignCertificate` response, requiring no API changes. This enables Dapr workloads to use JWTs for federated cloud credentials and infrastructure access.

* Areas affected: Dapr Sentry, Dapr runtime, Dapr components
* Proposed: Add JWT and OIDC support to Sentry, with configuration for signing keys, algorithms, JWKS, and OIDC server options

## Background

Dapr currently issues x.509 SVIDs for workload identity using SPIFFE. Many cloud providers and third-party systems only support federated authentication using JWTs and OIDC discovery endpoints.
Supporting JWTs in Sentry enables Dapr workloads to:
- Use JWTs as credentials for identity provider federation

This proposal addresses the need for flexible, standards-based workload identity in Dapr, improving security and interoperability for cloud-native and hybrid environments.

## Related Items

### Related proposals
- [20231024-CIR-trust-distribution.md](20231024-CIR-trust-distribution.md): Trust bundle and CA distribution

### Related issues

## Expectations and alternatives

* In scope:
  - Sentry issues JWTs as workload credentials, signed with configurable key/algorithm
  - JWKS endpoint for trust distribution
  - Optional OIDC discovery server
  - Audience claim configurable per app ID (default: trust domain)
  - JWT rotation mechanism
* Not in scope:
  - Full OIDC authorization flows (only discovery and JWKS endpoints)
  - Custom claims beyond audience, issuer, and standard fields
* Alternatives considered:
  - Only supporting x.509: limits interoperability with cloud and OIDC systems
  - External JWT/OIDC proxy: adds operational complexity, not integrated with Sentry
* Trade-offs:
  - Adds complexity to Sentry configuration and deployment
  - Exposes unauthenticated endpoints (user must secure as needed)
* Advantages:
  - Enables modern, federated identity scenarios
  - No breaking changes to existing APIs
* Disadvantages:
  - More configuration required for Sentry
  - Responsibility for endpoint security is on the user

## Implementation Details

### Design

- Sentry configuration extended to accept JWT signing key, algorithm, and JWKS (or auto-generate if not provided)
- JWT issued alongside x.509 SVID in `SignCertificate` response
- Audience claim set from app ID configuration (default: trust domain)
- JWKS endpoint served for public key distribution
- Optional OIDC discovery server serves `.well-known/openid-configuration` and `jwks.json` endpoints
- OIDC server exposed via Kubernetes Service; user responsible for external exposure and protection
- JWT rotation uses same mechanism as x.509 rotation
- Implementation extends existing SPIFFE package to support JWT-SVIDs
- daprd runtime and components can use JWT for cloud federation via token provider

### Feature lifecycle outline

* Expectations: Feature is opt-in, backward compatible, and does not affect existing x.509 flows
* Compatibility guarantees: No breaking changes to Sentry or workload APIs
* Deprecation/co-existence: x.509 and JWT can be issued together; no deprecation
* Feature flags: Enable/disable JWT and OIDC server via Sentry configuration

### Acceptance Criteria

* Sentry issues valid JWTs with correct claims and signatures
* JWKS endpoint serves correct public keys
* OIDC discovery endpoint is standards-compliant
* JWTs can be used for cloud provider federation
* No regression in x.509 SVID issuance or trust distribution

## Completion Checklist

* Code changes in Sentry for JWT/OIDC support
* Configuration and documentation updates
* Tests for JWT issuance, JWKS, and OIDC endpoints
* SDK/component changes to consume JWT as credential (Azure)

