# Serverless compute target

* Author(s): @ItalyPaleAle
* State: Draft
* Updated: 2022-12-09

## Overview

This is a proposal for being able to start a Dapr runtime ("daprd") as a proxy to **serverless compute** resources, a collective term that includes things like Azure Functions, AWS Lambda, Cloudflare Workers (and workerd), OpenFaaS, etc.

The goal is to allow developers to build apps that can leverage serverless compute in a smooth way, transparently integrating with Dapr. It allows building apps that can leverage flexible, on-demand compute resources, as a target for Dapr service invocation, PubSub messages, or input bindings.

This will be implemented by allowing Dapr to use a new class of components, called "compute", as a target for Dapr API calls, instead of an app running side-by-side with the Dapr runtime.

## Background

Serverless compute offers a programming model that allows developers to run code (usually written in JavaScript/TypeScript or compiled to WebAssembly) inside a "serverless runtime", which includes cloud-hosted services (Azure Functions, AWS Lambda, Cloudflare Workers, etc) as well as self-hosted projects (workerd, OpenFaaS). The main benefit to developers is that it expose compute resources that are flexible, available on-demand, and "infinitely scalable", so they can also tolerate workloads that are highly burstable.

With this proposal, we will make it easier for developers to add code running on serverless compute and integrate that seamlessly in an architecture that is based on Dapr. Developers will be able to make service invocation calls from other Dapr-ized apps that are served by serverless compute, as well as use serverless compute resources for processing PubSub messages and input bindings via Dapr.

## Expectations and alternatives

* **What is in scope for this proposal?**
  - This is a proposal for starting a Dapr sidecar that targets a serverless app for all messages sent from Dapr to the app. It will allow creating pods where only daprd is running and targets a serverless function for service invocation and for delivering PubSub messages and input bindings.
  - For each serverless platform we support, we will need to create either a "SDK" or a "gateway function" (depending on the capabilities of the platform) that will take care of the "plumbing" to allow Dapr to invoke the serverless code the developers write, taking care of things like authentication (see the "implementation details" section for more details).
* **What is deliberately *not* in scope?**
  - This is not a proposal for integrating Dapr in a serverless environment.
  - Although this will allow Dapr to invoke an app that is running as serverless, it does not allow the app to call into the Dapr APIs. This is because it's expected that Dapr will reside in places such as a K8s cluster and will be behind a firewall, while serverless code often runs in managed cloud environments, so communication from the app to Dapr won't be possible. Additionally, only HTTP can be used for the app channel.
  - This is not a proposal for creating an abstraction layer that allows running code on any serverless platform in a portable way. Dapr will offer a portable way to invoke that code from other Dapr-ized apps, and to use code running on serverless to process PubSub messages and input bindings; however, developers are responsible for writing code in a way that is supported by their serverless platform of choice.
* **What alternatives have been considered, and why do they not solve the problem?**
  - Using the HTTP output binding already allows invoking serverless code, but that does not allow using that code for things like service invocation or PubSub messages.
* **What advantages / disadvantages does this proposal have?**
 - This is the initial foray into integrating Dapr with serverless and it should be considered as a first step only.

## Implementation Details

The implementation will involve a few steps:

- Creating a new type of components called **`compute`**
- Adding support in the runtime for using external compute components rather than a local app channel
- Creating the required "plumbing" in the serverless code

### "compute" components

These components reside in the components-contrib repo and implement the following interface:

```go
// Compute is an interface to an external compute provider.
type Compute interface {
	// Init the component
	Init(metadata Metadata) error

    // Features returns the list of supported features.
	Features() []Feature

	// OnInvoke is invoked as a result of service invocation.
    OnInvoke(
        ctx context.Context, method string, data []byte, contentType string, metadata map[string]string,
    ) (
        statusCode int, data []byte, contentType string, metadataOut map[string]string, err error,
    )

    // OnTopicEvent is invoked as a result of an incoming PubSub message.
    OnTopicEvent(
        ctx context.Context, // other parameters TBD
    ) (
        statusCode int, err error,
    )

    // OnBindingEvent is invoked as a result of an incoming message from a binding.
    // The application can optionally return a response which is stored by the output binding if supported.
    OnBindingEvent(
        ctx context.Context, // other parameters TBD
    ) (
        any, error, // return values TBD
    )

    // ListTopicSubscriptions returns the list of topics  this serverless app subscribes to.
    ListTopicSubscriptions(ctx context.Context) (any, error) // return values TBD

    // ListInputBindings returns the list of input bindings this serverless app subscribes to.
    ListInputBindings(ctx context.Context) (any, error) // return values TBD
}

// Feature names a feature that can be implemented by compute components.
// (Currently no features are listed).
type Feature string
```

The APIs above map closely to the [AppCallback gRPC](https://github.com/dapr/dapr/blob/5d7dbfccaa792cebc694196acbc1ca513ba3feb4/dapr/proto/runtime/v1/appcallback.proto#L30) service that Dapr uses.

One important note is on the `Init` method, as there will be some expectations around it. It will need to:

- Check that the target serverless app is deployed
- Check that the target serverless app is running a supported version (more below)
- Check that requests against the target serverless app are successfully authorized (more below)

Compute components are defined in regular Dapr Component YAML files, for example:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: myserverless
spec:
  version: v1
  type: compute.azure.functions
  metadata:
    - name: endpoint
      value: "https://helloworld.azurewebsites.net/"
    - name: key
      # Include a private key either PEM-encoded or as a JSON string (JWK)
      value: "..."
```

### Configure daprd to use the external compute resource

In order for daprd to use the external compute resource, we will add a new property in the Configuration CRD. This works similarly to how we enable middleware components:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: serverless-dapr
spec:
  compute:
    # Name of the component defined in the component YAML
    name: myserverless
    # Type of the component
    type: compute.azure.functions
```

When launching daprd, we will use the existing `--config` and `--components-path` options to specify where to load the components from as well as the Configuration YAML to use.

Note that when using an external compute resource, daprd will _not_ initialize a local app channel, so options like `--app-port` and `--app-protocol` are ignored.

### The serverless code

Inside the code running the in the serverless platform, developers will be able to do pretty much whatever they want. As mentioned earlier, Dapr will not offer a "universal runtime" or "universal programming model" that works across all serverless runtimes, and we will not abstract the differences of the various platforms.

However, there is one thing Dapr will need to do for each supported runtime: offer a way to set up the required "plumbing", so that Dapr can invoke the serverless code. The most important thing is managing authentication and the "ping" messages.

This can be done in one of two ways:

- By offering a SDK (e.g. a NPM package) that developers will add to their own code. This will include methods that developers need to call to authenticate requests from Dapr as well as a way to return the version of the SDK (which Dapr invokes to make sure it's supported).
- By offering a "jump function", which is managed by Dapr and is tasked with invoking other functions the user manages (e.g. [Worker Services](https://developers.cloudflare.com/workers/learning/using-services/))

As mentioned, the components' `Init` methods are responsible for checking that the target serverless code exists and it's running a supported version (by invoking a `/.well-known/dapr/info` endpoint, for example). It will also ensure that authentication is successful.

For components that rely on "jump functions", Dapr can optionally manage the serverless code for the user. Take the exmaple of Cloudflare Workers: the `Init` method can check if the worker exists and is on the right version; if not, it will use the Cloudflare REST APIs to deploy the required worker.

As for authentication, that is performed via a "Bearer token" which uses the JWT standard. The token will include properties such as the name of the target serverless function and an expiration date. It will be signed with the private key defined in the component's metadata (using Ed25519 will be strongly recommended whenever possible); the serverless code will have the public part of the key that will use to verify the signature.

## Completion Checklist

* Components: at least 3 available at launch
* Code changes in the runtime
* Tests added (e2e, unit)
* Update the Dapr CLI
* Documentation

