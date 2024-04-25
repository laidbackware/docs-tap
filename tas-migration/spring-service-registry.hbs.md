# Migrate Spring Service Registry apps to Tanzu Application Platform

This topic tells you how to migrate Spring Service Registry apps from Tanzu Application Service
(commonly known as TAS) to Tanzu Application Platform (commonly known as TAP).
This topic uses the sample [Greeter app](https://github.com/spring-cloud-services-samples/greeting/tree/main)
in the procedures to illustrate how migration works.

Tanzu Application Service and Tanzu Application Platform can both provide apps with an
implementation of the Service Discovery pattern. You can migrate an app running on TAS with Spring
Service Registry (SSR) to Tanzu Application Platform without changing the app code.

At a high level, SSR on Tanzu Application Service and SSR on Tanzu Application Platform operate as follows:

- **For Tanzu Application Service:**

  Creating a `service-registry` instance in a TAS space creates a `cf` app running a Eureka server.

  TAS apps bound to this service instance are autowired with a Spring Cloud Netflix Eureka client
  configuration bean. The presence of the `VCAP_APPLICATION` environment variable triggers the bean
  to examine the `VCAP_SERVICES` environment variable to inject `eureka.client.*` properties into
  the app by using connection information in `VCAP_SERVICES`.

- **For Tanzu Application Platform:**

  `EurekaServer` instances are Kubernetes custom resources. Each
  `EurekaServer` instance is reconciled into a Kubernetes deployment running a Eureka server with a
  bindable `Secret` object containing connection information for that Eureka server.

  When a `EurekaServer` is wired into Tanzu Application Platform workloads using a `ResourceClaim`
  object, it mounts the bindable `Secret` object into the application container’s file system at a
  standard location.

  The [spring-cloud-bindings](https://github.com/spring-cloud/spring-cloud-bindings) library in
  GitHub finds any service bindings in this standard location and verifies that the binding is a
  Eureka server, based on a binding `type` key in the binding secret, and injects `eureka.client.*`
  properties into the app by using connection information in the mounted secret.

## <a id="migration-strategies"></a> Migration strategies

The recommended deployment pattern for Service Registry on TAS is to run a Service Registry for each high level business application, as Eureka the underlying technology in Service Registry is not built to be a business wide registry. As a result of this the expected topology on TAS will be a collection of Service Registries, each associated with a collection of TAS apps.

It is recommended to move all TAS apps associated with a single service registry at the same time. This is because it is not possible to peer replicate from the TAP based Service Registry back to the TAS based Service Registry and without this replication the TAS Service Registry cannot be complete.

If on TAS peer replication is used between Service Registry instances to share common services between many applications, then the TAP service registry must be exposed outside the cluster using a ### and the shared TAS Service Registry set to replicate towards the TAP registry. Then once all TAS apps have been migrated, then shared registry and it's apps can be migrated.

!! Open items
- How do you setup replication between TAP based service registries?
- Is the TAP SR HTTP based, so would allow exposing outside the TAP cluster via an ingress controller?
- Is any authN/Z possible on the TAP SR? If not this will likely rule it out for most customers.

## <a id="deploy-app-to-tas"></a> Deploy the `greeter` app to Tanzu Application Service

This section describes, at a high level, the steps for deploying an example `greeter` app
on Tanzu Application Service.

To deploy the app:

1. Create an instance of the service registry product from Marketplace by running:

   ```console
   cf create-service p.service-registry standard greeter-service-registry
   ```

1. Run `cf push` to push the apps to TAS with an app manifest that binds the service instance.
   Example:

    ```yaml
    ---
    applications:
    - name: greeter-cs
      instances: 1
      memory: 1G
      services:
      - greeter-service-registry
      env:
        SPRING_PROFILES_ACTIVE: development
        JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 17.+ }}'

    - name: greeter-messages-cs
      instances: 1
      memory: 1G
      services:
      - greeter-service-registry
      env:
        SPRING_PROFILES_ACTIVE: development
        JBP_CONFIG_OPEN_JDK_JRE: '{ jre: { version: 17.+ }}'
    ```

For more information about the Greeter app, see its
[README](https://github.com/spring-cloud-services-samples/greeting/blob/main/README.adoc#building-and-deploying-on-tanzu-application-service-tas)
file in GitHub.

For more information about Service Registry, see the
[TAS documentation](https://docs.vmware.com/en/Spring-Cloud-Services-for-VMware-Tanzu/3.2/spring-cloud-services/GUID-service-registry-index.html).

## <a id="migrate-tas-to-tap"></a> Migrate from Tanzu Application Service to Tanzu Application Platform

This section describes the areas to translate when creating a Tanzu Application Platform workload
that deploys the same app.

### <a id="create-db"></a>Create a database instance

Tanzu Application Service
: Create a service instance by running:

  ```console
  cf create-service
  ```

Tanzu Application Platform
: For Tanzu Application Platform, you must create a `EurekaServer` resource.

### <a id="bind-db"></a>Bind the database instance to the app

Tanzu Application Service
: Include the name of the service instance in the `services` key in the app manifest.
  Alternatively, use `cf bind-service` to bind the service to the app.

Tanzu Application Platform
: Create a `ResourceClaim` resource, specifying the details of the `EurekaServer` resource in the
  `spec.ref` section of the configuration.
  Specify the `ResourceClaim` resource in the `spec.serviceClaims` section of the `Workload`
      resource.

## <a id="deploy-app-to-tap"></a> Deploy the `greeter` app to Tanzu Application Platform

This section describes, at a high level, the steps for deploying an example `greeter` app
on Tanzu Application Platform.
For more information about how to deploy a service registry instance and configure workloads, see
[Overview of Service Registry](../service-registry/overview.hbs.md).

To deploy the app:

1. Install the Eureka Service Registry package by running:

   ```console
   tanzu package available list service-registry.spring.apps.tanzu.vmware.com --namespace tap-install

   tanzu package install service-registry \
   --package service-registry.spring.apps.tanzu.vmware.com \
   --version VERSION -n tap-install

   tanzu package installed get service-registry -n tap-install
   ```

1. Create a `EurekaServer` resource by applying the following YAML to your Kubernetes cluster:

    ```yaml
    ---
    apiVersion: service-registry.spring.apps.tanzu.vmware.com/v1alpha1
    kind: EurekaServer
    metadata:
      name: eurekaserver-sample
      namespace: my-apps
    spec:
      replicas: 2
    ```

   A successful `EurekaServer` resource has a `Ready` condition set to `true` and a
   `status.binding.name` field pointing to a secret containing connection information.

1. Claim credentials by using `ResourceClaim`.

    ```yaml
    ---
    apiVersion: services.apps.tanzu.vmware.com/v1alpha1
    kind: ResourceClaim
    metadata:
      name: eurekaserver-sample
      namespace: my-apps
    spec:
      ref:
        apiVersion: service-registry.spring.apps.tanzu.vmware.com/v1alpha1
        kind: EurekaServer
        name: eurekaserver-sample
        namespace: my-apps
    ```

1. Configure workloads. The [greeting app](https://github.com/spring-cloud-services-samples/greeting)
   is a sample workload in GitHub. Deploy both `greeter` and `greeter-messages` by using the
   following YAML. The YAML claims `EurekaServer` credentials by adding a `spec.serviceClaims`
   section to each workload.

    ```yaml
    ---
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: greeter-messages
      namespace: my-apps
      labels:
        apps.tanzu.vmware.com/workload-type: server
        apps.tanzu.vmware.com/has-tests: "true"
        app.kubernetes.io/part-of: greeter
    spec:
      build:
        env:
          - name: BP_JVM_VERSION
            value: "17"
          - name: BP_GRADLE_BUILT_MODULE
            value: "greeter-messages"
          - name: BP_GRADLE_BUILD_ARGUMENTS
            value: "--no-daemon clean bootJar"
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "development"
      serviceClaims:
        - name: eureka
          ref:
            apiVersion: services.apps.tanzu.vmware.com/v1alpha1
            kind: ResourceClaim
            name: eurekaserver-sample
      source:
        git:
          url: https://github.com/spring-cloud-services-samples/greeting
          ref:
            branch: main
    ---
    apiVersion: carto.run/v1alpha1
    kind: Workload
    metadata:
      name: greeter
      namespace: my-apps
      labels:
        apps.tanzu.vmware.com/workload-type: web
        apps.tanzu.vmware.com/has-tests: "true"
        app.kubernetes.io/part-of: greeter
    spec:
      build:
        env:
          - name: BP_JVM_VERSION
            value: "17"
          - name: BP_GRADLE_BUILT_MODULE
            value: "greeter"
          - name: BP_GRADLE_BUILD_ARGUMENTS
            value: "--no-daemon clean bootJar"
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "development"
      serviceClaims:
        - name: eureka
          ref:
            apiVersion: services.apps.tanzu.vmware.com/v1alpha1
            kind: ResourceClaim
            name: eurekaserver-sample
      source:
        git:
          url: https://github.com/spring-cloud-services-samples/greeting
          ref:
            branch: main
    ```

## <a id="service-registry"></a> Use Service Registry with an executable JAR file app

In the `greeting` app example, `BP_GRADLE_BUILD_ARGUMENTS` is set to include the `bootJar` task in
addition to the default Gradle build arguments. This setting is necessary for this example base
because the `build.gradle` file contains a `jar` section and the Spring Boot buildpack does not
inject the `spring-cloud-bindings` library into the app if it is an executable JAR file.

`spring-cloud-bindings` is required to process the `serviceClaim` into properties that tell the
discovery client how to find the Eureka server.

To use Service Registry with an executable JAR file app, explicitly include
[spring-cloud-bindings v1.13.0](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-bindings/1.13.0)
or later and set the `org.springframework.cloud.bindings.boot.enable=true` system property as
described in the [library README file](https://github.com/spring-cloud/spring-cloud-bindings#spring-boot-configuration)
in GitHub.
