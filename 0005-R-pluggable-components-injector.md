# Pluggable components injector

- Author(s): Marcos Candeia (@mcandeia)
- State: Ready for Implementation
- Updated: 11/21/2022

## Overview

Pluggable components are components that are not included as part of the runtime, as opposed to built-in ones that are included. The major difference between pluggable components and built-in components is the operational burden related to bootstrap/start the pluggable component process that are not necessary when using a built-in one since they run in the same process as Dapr runtime. This operational burden is present in many ways when using pluggable components and can lead to errors and hard debugging. In addition, there are certain configurations that are tied to the Dapr and how the runtime registers the pluggable component that is repetitive and can be better handled by Dapr instead of delegating this responsibility to the end-user. This proposal suggest the addition of a new mode of execution for selected pluggable components: injectable pluggable components.

## Background

#### Decrease the operational burden

Even considering the new pluggable components annotation from [#5402](https://github.com/dapr/dapr/issues/5402), setting up applications to properly work with pluggable components still not an easy task due to the operational related to bootstrapping containers over and over again for each application that the user needs, especially if you consider that components are often not well [scoped](https://docs.dapr.io/operations/components/component-scopes/). Without scope, a component make itself available for all applications within the same namespace, meaning that every deployment/pod should re-do the same manual job of mounting volumes, declaring environment variables and pinning container images.

So let's say you have an application named `my-app` and, another one named `my-app-2`, your two deployments/pods will look like the following:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: app
labels:
  app: app
spec:
replicas: 1
selector:
  matchLabels:
    app: app
template:
  metadata:
    labels:
      app: app
    annotations:
      dapr.io/pluggable-components: "component"
      dapr.io/app-id: "my-app"
      dapr.io/enabled: "true"
  spec:
    volumes:
      - name: my-component-required-volume
        emptyDir: {}
    containers:
      - name: my-app
        image: my-app-image:latest
      ### This is the pluggable component container.
      - name: component
        image: component:v1.0.0
        volumes:
          - name: my-component-required-volume
            mountPath: "/my-data"
        env:
          - name: MY_ENV_VAR_NAME
            value: MY_ENV_VAR_VALUE

---
apiVersion: apps/v1
kind: Deployment
metadata:
name: app-2
labels:
  app: app-2
spec:
replicas: 1
selector:
  matchLabels:
    app: app-2
template:
  metadata:
    labels:
      app: app-2
    annotations:
      dapr.io/pluggable-components: "component"
      dapr.io/app-id: "my-app-2"
      dapr.io/enabled: "true"
  spec:
    volumes:
      - name: my-component-required-volume
        emptyDir: {}
    containers:
      - name: my-app-2
        image: my-app-2-image:latest
      ### This is the pluggable component container.
      - name: component
        image: component:v1.0.0
        volumes:
          - name: my-component-required-volume
            mountPath: "/my-data"
        env:
          - name: MY_ENV_VAR_NAME
            value: MY_ENV_VAR_VALUE
```

Notice that everything related to the pluggable component container is repeated, and if you have a third application that doesn't require your pluggable component to work, so you have to scope your component to be initialized with only these two declared deployments/pods.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-component
spec:
  type: state.my-component
  version: v1
  metadata: []
scopes:
  - "my-app"
  - "my-app-2"
```

For each deployment that you have to add in your cluster, if that requires such pluggable component, you must also add in the scope list of the component spec, which ends up being error prone and intrusive.

#### Component spec atomicity/self-contained

Allow interchangeable/swappable components are one of the top amazing features that we provide, with that, a user can, in runtime, swap out a component with the same interface for another. Pluggable components made this behavior more difficult to maintain as it requires coordination, for a small time window, the user must provide a way to Dapr access both components at same time, otherwise it becomes very difficult to orchestrate that change manually.
To exemplify, suppose that we want to replace the Redis PubSub with the Kafka PubSub, and they are pluggable components. This is not only a matter of replacing the component spec itself, but it will require orchestrating the related deployments, otherwise it would lead in having an application pointing out to Kafka but with no Kafka pluggable component running and vice-versa.

The following diagram is exemplifying how that orchestrated change must applied:

<img width="466" alt="image" src="https://user-images.githubusercontent.com/5839364/201184828-d4e7357b-716a-4a3b-b7a5-dd22d1be7cda.png">

> That can't be avoided in scenarios where Dapr is not present as an orchestrator, for instance, self-hosted mode, but there are platforms that supports extensibility for orchestrating applications and its dependencies, like Kubernetes.

re: You can argue that Kubernetes solve this scenario by reconciling the cluster state until it succeeds, but still, it severely degrade the user experience when requires additional knowledge to build their applications with Dapr.

## Related Items

### Related proposals

[Pluggable components Annotations](https://github.com/dapr/dapr/issues/5402)

### Related issues

N/A

## Expectations and alternatives

### What is in scope for this proposal?

This proposal aims to add a new execution mode for pluggable components, the dapr-injected pluggable components.

### What is deliberately _not_ in scope?

This proposal does not aims to manage users' pluggable components code. The goal here is to provide a better UX when using pluggable components while decrease the operation burden.

## Implementation Details

### Design

This proposal aims to add a new execution mode for pluggable components, the dapr-injected pluggable components, that makes the operational behind remarkable like the built-in components. The operational burden is still present somewhere but divided into small reusable pieces.

<meta charset="utf-8"><b style="font-weight:normal;" id="docs-internal-guid-69570229-7fff-318a-575c-cff928d2ef5b"><p dir="ltr" style="line-height:1.38;background-color:#ffffff;margin-top:0pt;margin-bottom:0pt;"><span style="font-size:11pt;font-family:Arial;color:#000000;background-color:transparent;font-weight:400;font-style:normal;font-variant:normal;text-decoration:none;vertical-align:baseline;white-space:pre;white-space:pre-wrap;">&nbsp;</span></p><div dir="ltr" style="margin-left:0pt;" align="left">

| Type              | Injected by Dapr                                                                         | Managed by User/Unmanaged                                                                                           |
| ----------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Configuration     | Dapr Injects env vars and mount the shared volumes                                       | The user manually mounts and declares shared volumes                                                                |
| Container updates | Dapr automatically detects and applies, rolling out changes based on declared components | Users must redeploy their applications with the new desired version                                                 |
| Persona           | Cluster operator/End user                                                                | End user                                                                                                            |
| Scope             | Does not need to be scoped                                                               | If not scoped, all applications should have deployed the pluggable component, otherwise runtime errors might happen |

</div></b>

#### Component spec annotations

The component spec is still the entry point for all component types being pluggable or not, given that the pluggable components are a subset of all users declared components, even more, the pluggable components can be inferred from the declared components, we can actually leverage that property to extend our component spec, by adding custom annotations to allow Dapr to inject the component container at the time the Injector is also injecting the Dapr sidecar container.

Example:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-component
  annotations:
    dapr.io/component-container-image: "component:v1.0.0"
spec:
  type: state.my-componentƒ
  version: v1
  metadata: []
```

Optionally you can mount volumes and add env variables into the containers by using the `dapr.io/component-container-volume-mounts(-rw)` and `dapr.io/component-container-env` annotations.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-component
  annotations:
    dapr.io/component-container-image: "component:v1.0.0"
    dapr.io/component-container-volume-mounts: "volume-name:/volume-path,volume-name-2:/volume-path-2" # read-only, "$VOLUME_NAME:$VOLUME_PATH,$VOLUME_NAME_2:$VOLUME_PATH2"
    dapr.io/component-container-volume-mounts-rw: "volume-name-rw:/volume-path-rw,volume-name-2-rw:/volume-path-2-rw" # read-write "$VOLUME_NAME:$VOLUME_PATH,$VOLUME_NAME_2:$VOLUME_PATH2"
    dapr.io/component-container-env: "env-var=env-var-value,env-var-2=env-var-value-2" #optional "$ENV_NAME=$ENV_VALUE,$ENV_NAME_2=$ENV_VALUE_2"
spec:
  type: state.my-component
  version: v1
  metadata: []
```

By default the injector creates undeclared volumes as `emptyDir` volumes, if you want a different volume type you should declare it by yourself in your pods.

#### Pod annotations

In order to allow users to turn off the component injector for their pod, a new annotation will be available, similar to the one that we have for enabling dapr: `dapr.io/inject-pluggable-components:"true"`. Let's rewrite the previous examples using the injected pluggable components feature, it would be something like:

The apps deployments/pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: app
labels:
  app: app
spec:
replicas: 1
selector:
  matchLabels:
    app: app
template:
  metadata:
    labels:
      app: app
    annotations:
      dapr.io/inject-pluggable-components: "true"
      dapr.io/app-id: "my-app"
      dapr.io/enabled: "true"
  spec:
    containers:
      - name: my-app
        image: my-app-image:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: app-2
labels:
  app: app-2
spec:
replicas: 1
selector:
  matchLabels:
    app: app-2
template:
  metadata:
    labels:
      app: app-2
    annotations:
      dapr.io/inject-pluggable-components: "true"
      dapr.io/app-id: "my-app-2"
      dapr.io/enabled: "true"
  spec:
    containers:
      - name: my-app-2
        image: my-app-2-image:latest
```

And the component spec:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: my-component
  annotations:
    dapr.io/component-container-image: "component:v1.0.0"
    dapr.io/component-container-volume-mounts: "my-component-required-volume;/my-data"
    dapr.io/component-container-env: "MY_ENV_VAR_NAME;MY_ENV_VAR_VALUE"
spec:
  type: state.my-component
  version: v1
  metadata: []
```

### Feature lifecycle outline

#### Expectations

The feature is expected to be delivered as part of dapr/dapr v1.10.0 as a preview feature together with the new pluggable components SDK.

#### Compatability guarantees

Pluggable components that has been used will not be affected by this.

#### Deprecation / co-existence with existing functionality

N/A

### Acceptance Criteria

N/A

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

- [] Change the sidecar injector to make requests to the operator for listing components (or list it using its own role)
- [] Add 1 more annotation for pods `dapr.io/inject-pluggable-components: "true"` and 3 more for components `dapr.io/component-container-image`, `dapr.io/component-container-env` and `dapr.io/component-container-volume-mounts`
- [] Add the components container injector based on declared components
