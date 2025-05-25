# Sentry JWT and OIDC Support for Workload Identity

* Author(s): Joni Collige
* State: Implemented
* Updated: 2025-05-24

## Overview

This proposal introduces support for Dapr Sentry to issue JWTs as an additional credential alongside x.509 certificates. JWTs are signed with a configurable key and algorithm, and a JWKS endpoint is provided for trust distribution. Optionally, Sentry can host a minimal OIDC discovery server, exposing `.well-known/openid-configuration` and `jwks.json` endpoints for third-party trust federation. JWTs are returned in the existing `SignCertificate` response, requiring no API changes. This enables Dapr workloads to use JWTs for federated cloud credentials and infrastructure access.

## Background

Dapr currently uses x.509 SVIDs for workload identity via SPIFFE. However, many cloud providers and third-party systems require JWTs and OIDC discovery for federated authentication. Adding JWT support to Sentry enables Dapr workloads to participate in these modern identity flows, improving interoperability and security in cloud-native and hybrid environments.

## Related Items

### Related proposals
- [20231024-CIR-trust-distribution.md](20231024-CIR-trust-distribution.md): Trust bundle and CA distribution

## Expectations and alternatives

**In scope:**
- Sentry issues JWTs as workload credentials, signed with configurable key/algorithm
- JWKS endpoint for trust distribution
- Optional OIDC discovery server
- Audience claim configurable per app ID (default: trust domain)
- JWT rotation mechanism

**Not in scope:**
- Full OIDC authorization flows (only discovery and JWKS endpoints)
- Custom claims beyond audience, issuer, and standard fields

**Alternatives considered:**
- Only supporting x.509: limits interoperability with cloud and OIDC systems
- External JWT/OIDC proxy: adds operational complexity, not integrated with Sentry

**Trade-offs:**
- Increases Sentry configuration and deployment complexity
- Exposes unauthenticated endpoints (user must secure as needed)

**Advantages:**
- Enables federated identity scenarios and cloud provider integration
- No breaking changes to existing APIs

**Disadvantages:**
- Additional configuration required for Sentry
- Endpoint security is the user's responsibility

## Implementation Details

### Design

- Sentry configuration extended for JWT signing key, algorithm, JWKS (auto-generated if not provided)
- JWT issued alongside x.509 SVID in `SignCertificate` response
- Audience claim set from app ID configuration (default: trust domain)
- JWKS endpoint for public key distribution
- Optional OIDC discovery server serves `.well-known/openid-configuration` and `jwks.json`
- OIDC server exposed via Kubernetes Service (user responsible for exposure and protection)
- JWT rotation uses same mechanism as x.509 rotation
- SPIFFE package extended to support JWT-SVIDs
- daprd runtime and components can use JWT for cloud federation via token provider

#### Proposed SignCertificateResponse

```go
// SignCertificateResponse is returned by Sentry's SignCertificate API.
type SignCertificateResponse struct {
    // Existing x.509 SVID fields
	WorkloadCertificate []byte
	TrustChainCertificates [][]byte
	ValidUntil             *timestamppb.Timestamp

	// JWT token for authentication. Always provided by Sentry if configured to issue JWTs.
	Jwt *string
}
```

#### Proposed token provider for components-contrib

```go
// Token provider function to retrieve JWT from SPIFFE context
tokenProvider := func(ctx context.Context) (string, error) {
    // New method on spiffecontext to fetch JWT SVID
	tknSource, ok := spiffecontext.JWTFrom(ctx)
	if !ok {
		return "", fmt.Errorf("failed to get JWT SVID source from context")
	}
	jwt, err := tknSource.FetchJWTSVID(ctx, jwtsvid.Params{
		Audience: AzureADTokenExchangeAudience,
	})
	if err != nil {
		return "", fmt.Errorf("failed to get JWT SVID: %w", err)
	}
	return jwt.Marshal(), nil
}
```

### Proposed Configuration

Audiences will be read by Sentry from the app's configuration, allowing each application to specify its own audience claim for the JWT. If not specified, the default audience will be the trust domain.

```go
// ConfigurationSpec is the spec for a configuration.
type ConfigurationSpec struct {
    // Existing fields...

	// +optional
	AccessControlSpec *AccessControlSpec `json:"accessControl,omitempty"`
}

// AccessControlSpec is the spec object in ConfigurationSpec.
type AccessControlSpec struct {
    // Existing fields...

	// +optional
	Audiences []string `json:"audiences,omitempty" yaml:"audiences,omitempty"`
}
```

### Proposed Sentry Configuration Flags

**JWT-related flags:**
- `--jwt-enabled` (bool): Enable JWT token issuance by Sentry (default: `false`)
- `--jwt-key-filename` (string): JWT signing key filename (default: `jwt.key`)
- `--jwks-filename` (string): JWKS (JSON Web Key Set) filename (default: `jwks.json`)
- `--jwt-issuer` (string): Issuer value for JWT tokens (no issuer if empty)
- `--jwt-signing-algorithm` (string): Algorithm for JWT signing, must be supported by signing key. (default: `RS256`, options: https://github.com/lestrrat-go/jwx/blob/3430caec7d1283f2f95de5e065aed1eaba47bc32/jwa/signature_gen.go#L18)
- `--jwt-key-id` (string): Key ID (kid) for JWT signing (default: base64 encoded SHA-256 of the key)

**OIDC-related flags:**
- `--oidc-server-listen-port` (int): Port for the OIDC HTTP server (0 for random, `nil` for disabled)
- `--oidc-server-listen-address` (string): Address for the OIDC HTTP server (default: `localhost`)
- `--oidc-server-tls-enable` (bool): Serve OIDC HTTP with TLS (default: `true`)
- `--oidc-server-tls-cert-file` (string): TLS certificate file for OIDC HTTP server (required if `oidc-server-tls-enable` enabled)
- `--oidc-server-tls-key-file` (string): TLS key file for OIDC HTTP server (required if `oidc-server-tls-enable` enabled)
- `--oidc-jwks-uri` (string): Custom URI for external JWKS access (default: `nil`)
- `--oidc-path-prefix` (string): Path prefix for all OIDC HTTP endpoints (default: `nil`)
- `--oidc-domains` (string slice): Allowed domains for OIDC HTTP endpoint requests (default: `nil`)

### Propose Key Management

If no signing key or JWKS is provided, Sentry will automatically generate a new RSA key and JWKS on startup. The JWKS will be served at the `/jwks.json` endpoint, and the JWT signing key will be used for issuing JWTs.
The existing mechanism for generating x.509 certificates will be extended to support the generation of the JWT signing key and JWKS. The generated signing key will use the RS256 algorithm by default as this is the most widely supported and avoids incompatibility issues when federating with cloud providers and third-party systems.
The user can also provide a pre-generated signing key and JWKS file, which Sentry will use instead of generating them.
The JWKS can be used to verify JWTs signed by multiple keys but Sentry can only sign JWTs with one key at a time. It is the user's responsibility to manage key rotation and ensure the JWKS is updated accordingly.
In order to rotate the JWT signing key in a backward-compatible way, the user must provide a JWKS that contains the public keys of any previously used signing keys and the new key. This allows clients to verify old JWTs against previous keys and new JWTs against the new key.

### Feature lifecycle outline

- Feature is opt-in, backward compatible, and does not affect existing x.509 flows
- No breaking changes to Sentry or workload APIs
- x.509 and JWT can be issued together; no deprecation
- Feature flags enable/disable JWT and OIDC server

### Acceptance Criteria

- Sentry issues valid JWTs with correct claims and signatures
- JWKS endpoint serves correct public keys
- OIDC discovery endpoint is standards-compliant
- JWTs can be used for cloud provider federation
- No regression in x.509 SVID issuance or trust distribution

## Completion Checklist

- Code changes in Sentry for JWT/OIDC support
- Configuration and documentation updates
- Tests for JWT issuance, JWKS, and OIDC endpoints
- SDK/component changes to consume JWT as credential

