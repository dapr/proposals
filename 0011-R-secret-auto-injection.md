# Title of proposal 

* Author(s): Steve P.
* State: Ready for Implementation
* Updated: 23/11/2022

## Overview

A brief description of the proposal; include information such as:

* What areas are affected by this change? ==> Runtime & eventually SDKs
* What is being proposed in this document? ==> This proposal aims to simplify the secret usage in any app configuration without any code change and without calling the Dapr API neither at app level.

## Background

The Area is about Secret Management.
I create this proposal following my discussion with the Dapr maintainers.

- I started to look at the DB Bindings such as the [PostgreSQL](https://docs.dapr.io/reference/components-reference/supported-bindings/postgres/) and saw it asks the Dev. to write SQL statements whereas any Dev. would prefer to use an ORM framework
- Dapr purpose is not to recreate a new ORM which I can understand so then I started to think about Dapr to support any existing ORM:
  - For Java: JPA, Hibernate, integration with Spring Boot / spring-data-jpa
  - For MicroProfile : it means Jakarta EE / JTA integration with Dapr
  - For MicroProfile / Quarkus : it would mean integration with Panache, Hibernate & spring-data-jpa
  - Entity Framework for .Net
  - Django for Python

- The challenge is that any ORM configuration will end up with a DB password in clear text or as an environment variable in a property file.

The purpose of this proposal is to solve that security problem finding a way for Dapr to automatically inject that DB password secret or more generally any kind of secrets into the App property **without any code change** at App level.

There are hundreds of use cases where Dev. need to use passwords in property files: trust store, key store, SMTP gateway, LDAP, MongoDB, REDIS, publish/subscribe, Kafka, RabbitMQ etc.)


__Spring Cloud already provides an abstraction layer of each Cloud Provider__

relevant examples for Secrets Managers :

- [Spring Cloud AWS](https://spring.io/projects/spring-cloud-aws#overview): does NOT support AWS secret Manager
- [Spring Cloud Azure](https://spring.io/projects/spring-cloud-azure): supports Spring Boot Starter for Azure Key Vault
    - See also [Azure SDK](https://github.com/Azure/azure-sdk-for-java/tree/main/sdk/spring/spring-cloud-azure-starter-keyvault-secrets) & [this Azure sampe](https://github.com/Azure-Samples/azure-spring-boot-samples/tree/spring-cloud-azure_v4.4.1/keyvault/spring-cloud-azure-starter-keyvault-secrets/property-source)

- [Spring Cloud GCP](https://spring.io/projects/spring-cloud-gcp): supports [Secrets Manager](https://googlecloudplatform.github.io/spring-cloud-gcp/3.4.0/reference/html/index.html#secret-manager), see this [Sample](https://github.com/GoogleCloudPlatform/spring-cloud-gcp/tree/main/spring-cloud-gcp-samples/spring-cloud-gcp-secretmanager-sample)


Workaround to Design Portable Apps: Define Cloud Providers in Maven Profiles so that :
- env=azure depends on com.azure.spring:spring-cloud-azure-starter-keyvault-secrets
- env=gcp depends on com.google.cloud:spring-cloud-gcp-starter-secretmanager

Limitations: not supported with AWS, not truly Portable 

Apps portability drawbacks: The Dev. would need to update the Maven Lib dependencies, rebuild & redeploy the project, so reversibility is OK but this is not truly directly portable ==> This is where I see Dapr could bring value to any framework, not only Java.


## Related Items

### Related proposals 

Links to proposals that are related to this (either due to dependency, or possibly because this will replace another proposal)
- [https://github.com/dapr/dapr/issues/5196](https://github.com/dapr/dapr/issues/5196)
- [https://github.com/quarkiverse/quarkus-dapr](https://github.com/quarkiverse/quarkus-dapr)

### Related issues 

- [https://github.com/dapr/dapr/issues/5548](https://github.com/dapr/dapr/issues/5548)
- [https://github.com/dapr/dapr/issues/3354](https://github.com/dapr/dapr/issues/3354)

## Expectations and alternatives

* What is in scope for this proposal?

    Application Secret Management integration with Dapr is in the scope.    

* What is deliberately *not* in scope?

    Updating the existing frameworks & ORM is out of scope.

* What alternatives have been considered, and why do they not solve the problem?
    
    I thought of other alternatives :
    1. Ask to each framework community/maintainers to update their framework to support Dapr regarding this secret injection scenario, this results in a Governance & Time Management challenge as the decision is completely out of control of the Dapr community/maintainers

    Basic use case example with Java Hibernate ORM
    ```sh
    hibernate.connection.url=jdbc:mysql://mydb42.mysql.database.azure.com:3306/myapp?useSSL=true
    hibernate.connection.username=adm_db
    hibernate.connection.password=ThisIsaHiddenSecret777!
    ```

    The Integration with Dapr could look like something simple like this:

    ```sh
    hibernate.connection.dapr.enabled=true
    hibernate.connection.dapr.secrets.bulk.enabled=false
    hibernate.connection.dapr.secret-store-name=mystore
    hibernate.connection.dapr.secret-name=HIBERNATE-CONNECTION-PASSWORD

    hibernate.connection.url=jdbc:mysql://mydb42.mysql.database.azure.com:3306/myapp?useSSL=true
    hibernate.connection.username=adm_db
    #hibernate.connection.password= ==> would be read by the OR Mframework calling Dapr

    ```
    If hibernate.connection.dapr.secrets.bulk.enabled=true then all properties including hibernate.connection.url, hibernate.connection.username and hibernate.connection.password would be automatically injected from KV secrets HIBERNATE-CONNECTION-URL,HIBERNATE-CONNECTION-USERNAME and HIBERNATE-CONNECTION-PASSWORD secrets

    Then the ORM framework would have to call the Dapr API /v1.0/secrets/myvault/HIBERNATE-CONNECTION-PASSWORD instead of just reading the hibernate.connection.password property

    This solution is not really suitable because there are thousands of frameworks and even "just" targetting the main ones such as Python Django, .Net Entity Framework, Java Hibernate, Spring-Data, Jakarta EE EJB, Quarkus would take years to get implemented.

    2. Passwordless Features but all platforms do not support it :
        - ex: no support on-prem, or with [Spring Cloud AWS](https://spring.io/projects/spring-cloud-aws#overview)
        - this is not clear to me regarding Spring Cloud GCP and the [JDBC Socket Factory](https://github.com/GoogleCloudPlatform/cloud-sql-jdbc-socket-factory) still requires a Password, supports JBBC only , not JPA
        - Spring Cloud Azure offer a very limited [Passwordless](https://aka.ms/delete-passwords) support for MySQL, PostgreSQL, SQL with JDBC, JPA, Kafka

* Are there any trade-offs being made? (space for time, for example)
Instead of having a code change at the App level, there would be more configuration chnage on Dapr side.

* What advantages / disadvantages does this proposal have? 
    - eventual disadvantage: the solution should run with Dapr both on K8S and on non K8S platforms, but I do not know if it would run on non K8S platforms (would require not to use a ConfigMap)
    - advantages: this is a universal solution that would work with any framework and any property, adressing thousands of use cases

## Implementation Details

### Design

How will this work, technically? Where applicable, include: 

* Design documents
* System diagrams
* Code examples

Basic use case example with Java Hibernate ORM
```sh
hibernate.connection.url=jdbc:mysql://mydb42.mysql.database.azure.com:3306/myapp?useSSL=true
hibernate.connection.username=adm_db
hibernate.connection.password=ThisIsaHiddenSecret777!
```

The Integration with Dapr could look like something simple like this:

```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: daprsecretcm
data:
  dapr.secret.auto-injection: enabled
  dapr.secret.auto-injection.bulk: disabled
  dapr.secret.auto-injection.secret-store-name: mystore
  dapr.secret.auto-injection.secret-name: HIBERNATE-CONNECTION-PASSWORD
  dapr.secret.auto-injection.language-type: java  # to be clarified to guess if that language supports properties like .Net, Java, Python  or Env. var ONLY for Rust or Go.
```

Dapr [Side car Injector](https://docs.dapr.io/concepts/dapr-services/sidecar/#kubernetes-with-dapr-sidecar-injector) would check 1 ConfigMap to auto inject Secrets in the App at runtime: Ex:  hibernate.connection.password is not set in the App properties file but Dapr know that he has to get a secret referenced by ‘dapr.secret.auto-injection.secret-name’ in the secret store referenced by ‘dapr.secret.auto-injection.store-name’

Design considerations to be clarified: How Dapr would inject that secret into the App into a consistent way for each language, for instance Go & Rust would support Env. variables only, not properties.

We could use [Spring naming conventions](https://docs.spring.io/spring-boot/docs/2.7.3/reference/html/features.html#features.external-config.files): if you use environment variables rather than system properties, most operating systems disallow period-separated key names, but you can use underscores instead (for example, SPRING_CONFIG_NAME instead of spring.config.name). 
See [Binding from Environment](https://docs.spring.io/spring-boot/docs/2.7.3/reference/html/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables) Variables for details :

To convert a property name in the canonical-form to an environment variable name you can follow these rules:
- Replace dots (.) with underscores (_).
- Remove any dashes (-).
- Convert to uppercase.

For example, the configuration property spring.main.log-startup-info would be an environment variable named SPRING_MAIN_LOGSTARTUPINFO.
Environment variables can also be used when binding to object lists. To bind to a List, the element number should be surrounded with underscores in the variable name.
For example, the configuration property my.service[0].other would use an environment variable named MY_SERVICE_0_OTHER.

There would be still challenges about the secret naming as described in the [Spring docs](https://microsoft.github.io/spring-cloud-azure/current/reference/html/index.html#special-characters-in-property-name):
This method can not satisfy requirement like spring.datasource-url. When you save spring-datasource-url in Key Vault, only spring.datasource.url and spring-datasource-url is supported to retrieve property value, spring.datasource-url isn’t supported. To handle this case, please refer to the following option: Use property placeholders.


### Feature lifecycle outline

* Expectations
* Compatability guarantees
* Deprecation / co-existence with existing functionality
* Feature flags

### Acceptance Criteria

How will success be measured? 

* Performance targets: N/A
* Compabitility requirements: the solution should be tested and compatible with at least
    - [Java SE / Hibernate / Spring Data](https://github.com/dapr/dapr/issues/5548#issuecomment-1323945665)
    - [ Java Micro Profile / Quarkus with panache](https://github.com/dapr/dapr/issues/5548#issuecomment-1323947255)
    - [.Net / Entity Framework](https://github.com/dapr/dapr/issues/5548#issuecomment-1323942143)
    - [Python Django ORM](https://github.com/dapr/dapr/issues/5548#issuecomment-1323941446)

* Metrics: N/A

## Completion Checklist

What changes or actions are required to make this proposal complete?

* Code changes
* Tests added (e2e, unit)
* Documentation
