# Dapr Trust Distribution

* Author(s): joshvanl
* State: Draft
* Updated: 2023-10-24

## Overview

This is a design proposal to implement a proper trust distribution process in Dapr.
Trust distribution will be implemented in a seamless way without downtime.
This will improve and unlock security related features.

## Background

### Motivation

To support the creation of features:

- Proper Certificate Authority (CA) rotation (without re-using the root's private key)
- External CA sources such as cert-manager and cloud provider CAs etc.
- Dapr multi-cluster and networking federation

Related issues:
- [Multicluster Kubernetes](https://github.com/dapr/dapr/issues/3460)
- [[Proposal] Support third-party CA - Integrate Cert Manager with Dapr](https://github.com/dapr/dapr/issues/3968)
- [[Proposal] Automatic root certificate rotation](https://github.com/dapr/dapr/issues/5958)

### Goals

- Implement an active trust distribution mechanism for Dapr in Kubernetes that responds to updates.
- Enable root rotation with no downtime.

### Non-Goals

- Sentry implements CA root rotation.
- Implement external CA support.
- Implement Dapr trust federation.

### Current Shortfalls

Trust distribution is the act of propagating trust data to enable secure communication or networking between peers.
In the case of Dapr, this involves propagating PEM encoded CA bundle files to clients and servers in the cluster, which are then used to authenticate peers over TLS.
Today, their are two methods of CA deployment to Dapr; either Sentry generated or provided by the Dapr cluster administrator.
From the prospective of trust distribution, these two modes are functionally the same as they both result in the `dapr-trust-bundle` ConfigMap containing the CA bundle, and the `dapr-trust-bundle` Secret containing the issuer certificate chain.
From here, trust distribution for the control plane occurs by the Operator, Placement, and Injector services reading from the mounted `dapr-trust-bundle` ConfigMap.
Trust distribution for Daprds occurs from the Injector patching the Daprd container with an environment variable containing the CA Bundle originating from the `dapr-trust-bundle` ConfigMap.

The problem with the current strategy is that once trust is distributed once (the `dapr-trust-bundle` ConfigMap and Secret is populated), the root of trust cannot change in the cluster.
This is because trust bundles are only injected to Dapr containers at Pod creation time.
Trust anchors are also set as environment variables whose values are static for the entire duration of a unix process, meaning they cannot be dynamically updated, for example in the event of CA root rotation.
Today, Dapr pods will have to be restarted in order to pick-up a new trust bundle.
It is also undefined and untested as to whether the control plane components will successfully pick-up a new trust bundle during execution; though whether they can is irrelevant as Daprds do not also support this feature.

## Solution

### Trust Distributor

It is paramount that the entity that conducts the trust distribution is separate from the entity that issues identities from that root of trust, in Dapr's case this is Sentry.
This is because trust distribution must happen out of band of identity issuance.
An analogous to this is roots of trust of the Internet are delivered via the computers Operating System or Internet Browser, rather than fetching them from DNS servers themselves.
Similarly for example, asset SHA hashes should be downloaded from a separate source then the assert server themselves.
Decoupling these roles also has the benefit of improving separation of concerns between responsibilities from the identity issuer, and the trust distributor.

Trust distribution will be conducted by the Operator and written to the ConfigMap `dapr-root-ca.crt` in all Namespaces.
The Operator is a natural fit as it is not Sentry (the identity issuance server), and machinery for Kubernetes controllers already exists in the Operator today.
ConfigMaps are a natural choice as they can be mounted by Pods & containers in Kubernetes, and trust bundles are not secrets so Secrets are not appropriate.
There is also prior art to other projects distributing trust in this way, such as [Istio](https://github.com/istio/istio/blob/4c65649a9b116584281fadcaf8c3dd6b42d34036/istioctl/pkg/workload/workload_test.go#L340) and cert-manager's [trust-manager](https://github.com/cert-manager/trust-manager#example-bundle).
We can also add support for writing to Kubernetes [ClusterTrustBundles](https://github.com/kubernetes/enhancements/issues/3257), though this resource is very new, and will not be available in all target Kubernetes cluster versions.
The `dapr-root-ca.crt` ConfigMap name is consistent with Kubernetes and Istio naming.
The operator will source the root of trust to be distributed from the mounted `dapr-root-ca.crt` Secret in the control plane namespace.
The operator will watch for this mounted file for updates using [fsnotify](https://github.com/fsnotify/fsnotify), and distribute the contents to the named ConfigMap in all namespaces.

Once propagated, the Injector, Placement, and Dapr sidecars can mount this ConfigMap and use it as the root of trust when connecting to peers.
Similarly, when the file is updated, these services can update their local trust stores to use the new version of the bundle.

The Operator will need to metadata watch all Namespaces and ConfigMaps in the cluster.
The Operator should no fully inform these resources as that will massively increase the memory consumption of the Operator.
In the event of a Namespace being created, the Operator will write the ConfigMap to that namespace.
The Operator will also ensure that the `dapr-root-ca.crt` ConfigMap stays consistent with its local trust bundle version.

### CA Rotation

CA rotation can now be solved by the new trust bundle being _appended_ to the existing `dapr-root-ca.crt` Secret in the control plane namespace.
This new bundle containing the old and new CA will be propagated to all services by the Operator, allowing for a zero downtime & graceful roll over of the CA.
The Dapr CLI will be updated to automate this task, ensuring that the new appended trust bundle has been correctly propergated to all namespaces before writing the new CA to sentry.
Checking propagation involves ensuring the new bundle contents is present at the named ConfigMap in all namespaces.
The CLI needs to take care of the fact that mounted ConfigMaps can take up to [60 seconds](https://github.com/kubernetes/kubernetes/blob/v1.26.0/pkg/kubelet/pod_workers.go#L1175C1-L1175C96) before the file is updated on the container mount, so there is some lag between the ConfigMap being updated and the trust bundle being updated in a service's trust store.

### External CA Support

External CA support is now made easier by the fact that the external CA trust bundle can be safely written to the `dapr-cert-ca.crt` Secret in the control plane namespace.
Dapr services will now trust the external CA's root of trust.

### Dapr multi-cluster and Networking Federation

Similarly, the trust bundle of another Dapr cluster can be appended to the existing CA bundle so that the two clusters may trust one another.

### Self Hosted Mode

Self hosted mode will continue to function as before, however using a file reference for the trust bundle rather than an environment variable means that services can respond and update their trust stores on file changes.

### Deprecation

The `DAPR_TRUST_ANCHORS` environment variable in Daprd will become deprecated, and instead favour using a file reference configured via the CLI flag `-trust-anchors-file`.
For backwards compoatabiliy, the `DAPR_TRUST_ANCHORS` environment variable will continue to be supported until `v1.14`, where the Injector service will no long patch it into Daprd sidecar containers.

## Completion Checklist

- [ ] In Kubernetes CA mode, Sentry writes its own generated CA bundle to the `dapr-root-ca.crt` Secret in the control plane namespace.
- [ ] The Operator propagates this trust bundle to the `dapr-root-ca.crt` ConfigMap in all namespaces.
- [ ] Placement, Injector, and Daprds all read and watch the trust anchors from the mounted `dapr-root-ca.crt` ConigMap referenced by the `-trust-anchors-file` flag's value, updating their trust stores accordingly.
- [ ] Dapr CLI CA rotation command updated to respect the `dapr-root-ca.crt` Secret and append the CA bundle accordingly.

### Acceptance Criteria
- Trust distribution is active and responds to updates.
- Dapr CLI `mtls renew-certificate` has been updated to implement proper CA rotation.
