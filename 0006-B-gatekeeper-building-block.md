# Title of proposal 

* Author(s): Xavier Geerinck
* State: Ready for Implementation
* Updated: 29/NOV/2022

## Overview

This proposal documents introduces the "Gatekeeper" building block which helps developers to implement "Authorization" requests for their application, assisting the developer by implementing core authorization requirements:
* Am I as a user able to access resource X?
* Which resources am I as a user able to access? (commonly known as an "ARQ" or "Access Request Query")

It works through added APIs in the Dapr runtime that work together with a state store to store the policies.

## Background

While working with software, one of the most common issues that pops-up is "Authorization". After a user is authenticated, how do we know that this user has access to a specific piece in your code? For this, developers typically reside towards implementing:
* RBAC (Role-based Access Systems)
* ABAC (Attribute-Based Access System)

The above is often considered a lengthy task, as users have to ensure that everything works as intended and test it carefully. A leak in this piece of code could have serious consequences.

Dapr runs as a sidecar architecture, typically close to the software. This software often interacts with users (Backend API) or as microservices doing an isolated piece of code.

Therefore, I would love to propose a new Building Block: **"Gatekeeper"**, where a user can simply call it through one of the Dapr SDKs and receive a response if they are allowed to access a certain feature or not.

### Vocabulary

* **Gatekeeper Principal:** Who is trying to access the resource and what are his roles? 
    * Interface: `{ id: string, roles: string[] }`
* **Gatekeeper Action:** What is the action the principal is trying to perform? (typically CREATE, READ, UPDATE, DELETE, MANAGE)
    * Interface: `string[]`
* **Gatekeeper Resource Kind:** What type of resource is the principal trying to access? (e.g., a user)
    * Interface: `string` (typically the Model names, e.g., User)
* **Gatekeeper Resource Type:** What resource is the principal trying to access?
    * Interface: `{ id?: string, kind: GatekeeperResourceKind, attr?: any }`


## Related Items

### Related proposals 

Links to proposals that are related to this (either due to dependency, or possibly because this will replace another proposal)

### Related issues 

Please link to any issues that this proposal is related to, for example, are there existing bugs filed in various Dapr repositories that this will affect?

* https://github.com/nextauthjs/next-auth/pull/5240
* https://github.com/dapr/dapr/issues/5380
* https://github.com/dapr/dapr/issues/5094

### Related Ecosystem Items

* https://cerbos.dev (heavily inspired upon)
* https://research.google/pubs/pub48190/ (Google Zanzibar: Global Authorization System)
* https://www.ory.sh/docs/keto

## Expectations and alternatives

* What is in scope for this proposal?

The proposal introduces 2 new endpoints that add if a user is able to access a resource and the resources the user can access. Next to these endpoints, the proposal also describes the implementation of a policy processor that is able to store and check policies

* What is deliberately *not* in scope?

The usage of different components, only one main gatekeeper will exist

* What alternatives have been considered, and why do they not solve the problem?

Implementation in the native SDKs, this is however a repetitive task that can be solved by implementing it in the core runtime

* Are there any trade-offs being made? (space for time, for example)

* What advantages / disadvantages does this proposal have? 

Through this proposal, developers will not have to worry anymore about their authorization system which is found to be a key focus point seeing the security risks it entails. Developers will now be able to focus on other business value critical opportunities instead of a task that has become repetitive over the years but stays to be implemented in a custom way.

## Implementation Details

### Design

How will this work, technically? Where applicable, include: 

* Design documents
* System diagrams
* Code examples

#### Endpoints

##### POST /v1.0/gatekeeper/check

Checks if the provided principal (e.g., user) is allowed to access the requested resource.

###### Parameters

* principal
* resource
* actions
* metadata

```javascript
{
    // Who is acting? (comes from Identity Provider)
    principal: {
        id: "my-user",
        roles: ["user"]
    },
    // Which resources are we accessing?
    resources: [
        resource: {
            kind: "user",
            
        },
        actions: ["read", "create"]
    ]
}
```

###### Example implementation

> 💡 This implementation is currently being used by myself as a test for a production system. It also shows how an Audit log could work

```typescript
async check(principal: GatekeeperPrincipalType, resource: GatekeeperResourceType, actions: AuthPermissionActionEnum[], throwError = false): Promise<boolean> {
  const ability = await this.abilityFactory.createForPrincipal(principal);

  let decision = false;

  if (!resource.id || !principal.id || !principal.roles) {
    decision = false;
  } else {
    actions.every(action => {
      if (resource.id && resource.attr) {
        return ability.can(action?.toLocaleLowerCase() as Lowercase<AuthPermissionActionEnum>, subject(resource.kind.toString(), resource.attr ?? {} as any));
      } else {
        return ability.can(action?.toLocaleLowerCase() as Lowercase<AuthPermissionActionEnum>, resource.kind.toString() as any);
      }
    });
  }

  // @todo: we could log this to an audit log
  console.log(`${actions.join(", ")} ${resource.kind} (id: "${resource.id}") ${decision ? "allowed" : "denied"} for Principal(id: "${principal.id}", roles: "${principal?.roles?.join(", ")}")`);

  if (!decision && throwError) {
    throw new Error(JSON.stringify({
      error: "ACCESS_DENIED",
      message: `Access to ${resource.kind} (id: "${resource.id}") was denied for Principal(id: "${principal.id}", roles: "${principal?.roles.join(", ")}")`
    }));
  }

  return decision;
}
```

##### POST /v1.0/gatekeeper/filter

Creates a filter (commonly called "ARQ" or "Access Request Query" to see what the user can access. In this case we create a filter that gets returned as a simple JSON to be transformed by the end-framework into the respective query language. 

###### Parameters

* Principal
* Resource

###### Output

Returns the filter describing what we are trying to access.

Example:

When we pass the principal `{ id: "MY-ID" }` for the resource `{ kind: "Book" }` we would get the return:

```javascript
{
    Book: {
        authorId: {
            eq: "MY-ID"
        }
    }
}
```

###### Example Implementation

```typescript
async createFilterByPrincipal(principal: GatekeeperPrincipalType, resource: GatekeeperResourceType): Promise<any> {
  const ability = await this.abilityFactory.createForPrincipal(principal);
  const filter = accessibleBy(ability)[resource.kind.toString()];

  // recreate the object to remove meta we don't need (accessibleBy formats it as a WhereInput)
  // whereas we just want the filter (e.g., { id: { in: [1,2,3] } })
  return JSON.parse(JSON.stringify(filter));
}
```

#### Dapr Implementation

Dapr would in this case utilize the State Store components to define the **Policies** of what a user can do. 

##### Example Policies

###### User

```javascript
{ 
    resource: GatekeeperResourceKindEnum.User, 
    actions: [AuthPermissionActionEnum.READ], 
    effect: AuthPermissionEffectEnum.ALLOW, 
    condition: { id: "${P.id}" } 
}
```

###### Admin

```javascript
{ 
    resource: GatekeeperResourceKindEnum.User, 
    actions: [AuthPermissionActionEnum.MANAGE], 
    effect: AuthPermissionEffectEnum.ALLOW, 
}
```

##### Example Dapr Gatekeeper Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
resourcePolicy:
  version: "default"
  resource: user
  rules:
    # Admins can do anything
    - actions: ["*"]
      effect: EFFECT_ALLOW
      roles:
        - admin
      
    # Users can read their own info
    - actions: ["read", "list"]
      effect: EFFECT_ALLOW
      roles:
        - owner
      condition:
        match:
          expr: ("userId" eq request.principal.id)
```

### Feature lifecycle outline

* Expectations

This feature will start as a preview feature to be stabilized later on

* Compatability guarantees

None

* Deprecation / co-existence with existing functionality

Collaboration with the State store functionality

* Feature flags

N/A

### Acceptance Criteria

How will success be measured? 

* Performance targets
* Compabitility requirements
* Metrics

## Completion Checklist

What changes or actions are required to make this proposal complete? Some examples:

* Code changes
* Tests added (e2e, unit)
* SDK changes (if needed)
* Documentation

This proposal requires more insights by others, mainly on the proposed REST APIs and their workings, translation into the Dapr runtime itself and core interest of users in the community.