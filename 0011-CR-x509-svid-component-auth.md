# Component X.509 SVID Authentication

* Author(s): Josh van Leeuwen (@joshvanl)
* Updated: 2024/01/12

## Overview

This proposal outlines how Components can use the Dapr SPIFFE identity X.509 certificate to authenticate to services.

## Background

### Motivation

Using X.509 SPIFFE identity certificates for authentication is advantageous for a number of reasons:

- Component metadata specs need not contain any secret data (e.g. passwords, tokens, etc.)
- Identity tied to the App ID for authentication means access policy is defined in the context of Dapr, rather than in the context of the target service primitives (e.g. Cloud Service Accounts).
- The target service also need not store or generate secret material for authentication, and instead store trust anchors for the Dapr SPIFFE identity CA.
- X.509 mTLS authentication has a strictly better security posture than password or token based authentication as no secrets are sent over the wire.

### Goals
- Enable X.509 SPIFFE identity certificate based authentication for Components.
- Enable X.509 Authentication to AWS Components using Roles Anywhere.

## Implementation

For Components to use SPIFFE X.509 certificate authentication, the Daprd SVID needs to be passed into components-contrib during a Component Init.
To do this, we can either pass the password and certificate chain via a well known internal metadata key or as a [Context value](https://pkg.go.dev/context#WithValue).
Both of these methods have pros and cons:

- Context values allows passing a pointer to the dynamic SVID meaning that the components-contrib will be able to obtain the latest SVID even after the orginal SVID has been renewed.
  Metadata values are static strings and therefore only the initial first SVID will be available to the component.

- Passing the SVID as a context value could potentially be risky since the value may be propagated or copied into other contexts leading to the SVID being leaked.
  Static metadata values do not have this problem as they are only available to the component.

In the initial implementation, I have passed the SVID as the metadata values:

```
__internal.dapr.io/svid-certificate-chain
__internal.dapr.io/svid-private-key
```

#### Example of implementation

https://github.com/dapr/kit/compare/main...JoshVanL:kit:crypto-pem

https://github.com/dapr/components-contrib/compare/main...JoshVanL:components-contrib:aws-auth-svid-x509

https://github.com/dapr/dapr/compare/master...JoshVanL:dapr:components-init-x509-svid
