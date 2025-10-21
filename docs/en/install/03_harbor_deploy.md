---
title: "Harbor Instance Deployment"
description: "Harbor is an open-source container registry platform that provides container image management, security scanning, and distribution capabilities."
weight: 200
---

This document describes the subscription of Harbor Operator and the functionality based on Harbor instance deployment.

## Prerequisites

- This document applies to Harbor 2.12 and above versions provided by the platform. It is decoupled from the platform based on technologies such as Operator.

- Please ensure that Harbor Operator has been deployed (subscribed) in the target cluster, meaning the Harbor Operator is ready to create instances.

## Deployment Planning

Harbor supports various resource configurations to accommodate different customer scenarios. In different scenarios, the required resources and configurations may vary significantly. Therefore, this section describes what aspects need to be considered in deployment planning before deploying Harbor instances, and what the impact of decision points is, to help users make subsequent specific instance deployments based on this information.

### Basic Information

1. The Harbor Operator provided by the platform is based on the community's official Harbor Operator, with enterprise-level capability enhancements such as ARM support and security vulnerability fixes. It is fully compatible with the community version in terms of functionality, and in terms of experience, it enhances the convenience of Harbor deployment through optional and customizable templates.

2. A Harbor instance contains multiple components, such as the `Registry component` responsible for managing image files, the `PostgreSQL` component providing storage for application metadata and user information, and the `Redis` component for caching, etc. The platform provides professional PostgreSQL Operator and Redis Operator, so when deploying Harbor instances, Redis and PostgreSQL resources are no longer directly deployed, but accessed through configuring specific access credentials for existing instances.

### Pre-deployment Resource Planning

Pre-deployment resource planning refers to decisions that need to be made before deployment and take effect during deployment. The main contents include:

**High Availability**

- Harbor supports high availability deployment, with the following main impacts and limitations:

   - Each component will use multiple replicas

   - Network access no longer supports `NodePort`, but requires access through domain names configured via `Ingress`

   - Storage methods no longer support `node storage`, but require access through `StorageClass` or `PVC`

**Resources**

According to community recommendations and practices, a non-high-availability Harbor instance can run with a minimum of 2 cores and 4Gi of resources, while in high availability mode, a minimum of 8 cores and 16Gi of resources is required for stable operation.

**Storage**

::: tip Storage Selection Guide
| Component | Supported Storage Type | Description |
|---| ---| ---|
| Registry | object storage (e.g., MinIO)<br >file storage (e.g., Ceph FS) | For scenarios with high pull image concurrency, it is recommended to use object storage and enable the redirect feature.<br />The sequential read/write throughput of the storage disk (with a 1 MB block size) should not be lower than 100 MiB/s. |
| JobService | file storage (e.g., Ceph FS) | Job logs can also be [stored in the database](../howto/07_configure_job_log_storage.mdx#store-job-log-to-db), in which case it is not necessary to configure storage for the job service. |
| Trivy | file storage (e.g., Ceph FS) | The Trivy component uses storage to save its `trivy db` and scan cache |

:::

- Common storage methods provided by the platform can be used for Harbor, such as storage classes, persistent volume claims, node storage, etc.

- Node storage is not suitable for `high availability` mode, as it stores files in a specified path on the host node

- Additionally, Harbor supports object storage. For configuration instructions, see [Using Object Storage as the Registry Storage Backend](#storage-yaml-snippets).

**Network**

- The platform provides two mainstream network access methods: `NodePort` and `Ingress`

   - `NodePort` requires specifying HTTP port and SSH port, and ensuring the ports are available. `NodePort` is not suitable for `high availability` mode

   - `Ingress` requires specifying a domain name and ensuring that the domain name resolution is normal

- The platform supports HTTPS protocol, which needs to be configured after instance deployment. See [Configure HTTPS](#harbor_https) for details.

**Redis**

::: tip Redis Component Storage Selection Guide
It is recommended to use block storage (e.g., TopoLVM) to achieve higher IOPS and lower latency.
:::

- The current Redis component version that Harbor depends on is v6. It is recommended to use the Redis Operator provided by the platform to deploy Redis instances, and then complete Redis integration by configuring access credentials.

   - Redis access is accomplished by configuring a `secret` resource with specific format content. See [Configure Redis, PostgreSQL, and Account Access Credentials](./02_harbor_credential) for details.

- **Known Issue**: [Modify Harbor Project Permissions Prompt Internal Server Error](../trouble_shooting/01_modify_project_permissions_error.mdx)

**PostgreSQL**

::: tip PostgreSQL Component Storage Selection Guide
It is recommended to use block storage (e.g., TopoLVM) to achieve higher IOPS and lower latency.
:::

- The current PostgreSQL component version that Harbor depends on is v14. It is recommended to use the PostgreSQL Operator provided by the platform to deploy PostgreSQL instances, and then complete PostgreSQL integration by configuring access credentials.

   - PostgreSQL access is accomplished by configuring a `secret` resource with specific format content. See [Configure Redis, PostgreSQL, and Account Access Credentials](./02_harbor_credential) for details.

**Account Credentials**

When initializing a Harbor instance, you need to configure the admin account and its password. This is done by configuring a `secret` resource. See [Configure Redis, PostgreSQL, and Account Access Credentials](./02_harbor_credential) for details.

### Post-deployment Configuration Planning

Post-deployment configuration planning refers to planning that does not require decisions before deployment, but can be changed as needed through standardized operations after deployment. This mainly includes Single Sign-On (SSO), HTTPS configuration, external load balancer configuration, etc. Please refer to [Subsequent Operations](#harbor_day2) for details.


## Instance Deployment

The Harbor Operator provided by the platform mainly offers two deployment methods: deployment from templates and deployment from YAML.

The platform provides two built-in templates for use: the `Harbor Quick Start` template and the `Harbor High Availability` template, while also supporting custom templates to meet specific customer scenarios.

Information about the built-in templates and YAML deployment is as follows:

### Deploying from the `Harbor Quick Start` Template

This template is used to quickly create a lightweight Harbor instance suitable for development and testing scenarios, not recommended for production environments.

- Computing resources: CPU 2 cores, Memory 4Gi
- Storage method: Uses local node storage, requires configuring storage node IP and path
- Network access: Uses NodePort method, shares node IP with storage, requires port specification
- Dependent services: Requires configuration of existing Redis and PostgreSQL access credentials
- Other settings: Requires account credentials configuration, SSO functionality is disabled by default

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from the `Harbor High Availability` Template

Deploying a high-availability Harbor instance requires higher resource configuration and provides a higher availability standard.

- Computing resources: CPU 16 cores, Memory 16 Gi
- Storage method: Uses storage class resources to store image files, background task logs, and image scanning vulnerability database
- Network access: Uses Ingress method, requires domain name specification
- Dependent services: Requires configuration of existing Redis and PostgreSQL access credentials
- Other settings: Requires account credentials configuration, SSO functionality is disabled by default

To achieve Harbor high availability, external dependencies must meet the following conditions:

1. The `Redis` and `PostgreSQL` instances must be highly available
2. The network load balancer must be highly available; when using ALB, a VIP must be configured
3. The cluster must have more than 2 nodes

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from YAML


YAML deployment is the most basic and powerful deployment capability. Here we provide corresponding YAML snippets for each dimension from the `Deployment Planning` section, and then provide two complete YAML examples for complete scenarios to help users understand the YAML configuration method and make configuration changes as needed.

#### High Availability (YAML Snippets)

In high availability mode, Harbor component replicas should be at least 2. The YAML configuration snippet is as follows:

```yaml
spec:
  helmValues:
    core:
      replicas: 2
    portal:
      replicas: 2
    jobservice:
      replicas: 2
    registry:
      replicas: 2
```

#### Storage (YAML Snippets)

Harbor data storage mainly includes two parts:

- Registry: Manages and stores container images and artifacts. It handles image upload, download, and storage operations.
- Jobservice: Executes background tasks like image replication between registries, garbage collection, and other scheduled or on-demand jobs.
- Trivy: Performs vulnerability scans on container images to identify security issues and ensure compliance with security policies.

Currently, three storage configuration methods are supported: storage class, PVC, and local node storage.
When using storage class or pvc, the storage must support multi-node read and write (ReadWriteMany)

Storage class configuration snippet:

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      persistentVolumeClaim:
        registry:
          storageClass: ceph
          accessMode: ReadWriteMany
          size: 10Gi
        jobservice:
          jobLog:
            storageClass: ceph
            accessMode: ReadWriteMany
            size: 1Gi
        trivy:
          storageClass: ceph
          accessMode: ReadWriteMany
          size: 5Gi

```

PVC configuration snippet (PVCs need to be created in advance):

```yaml
spec:
  helmValues:
    harbor:
      persistence:
        enabled: true
        persistentVolumeClaim:
          registry:
            existingClaim: <registry component pvc>
          jobservice:
            jobLog:
              existingClaim: <jobservice component pvc>
          trivy:
            existingClaim: <trivy component pvc>
```

Local node storage configuration snippet:

```yaml
spec:
  helmValues:
    persistence:
      enabled: true
      hostPath:
        registry:
          path: <registry component node storage path>
        jobservice:
          path: <jobservice component node storage path>
        trivy:
          path: <trivy component node storage path>
    registry:
      nodeSelector:
        kubernetes.io/hostname: <node name>
    jobservice:
      nodeSelector:
        kubernetes.io/hostname: <node name>
    trivy:
      nodeSelector:
        kubernetes.io/hostname: <node name>
```

Configure Object Storage (S3) as the Registry storage backend:

- The Object Storage bucket must be created in advance.
- The [`<object-storage-secret>`](./02_harbor_credential.md#object-storage-credentials) secret must be created in advance.

  ```yaml
  spec:
    helmValues:
      harbor:
        persistence:
          enabled: true
          imageChartStorage:
            disableredirect: true
            s3:
              existingSecret: <object-storage-secret>
              bucket: <bucket>
              region: <region>
              regionendpoint: <regionendpoint>
              v4auth: true
            type: s3
          persistentVolumeClaim:
            jobservice:
              jobLog:
                existingClaim: <jobservice component pvc>
            trivy:
              existingClaim: <trivy component pvc>
  ```

| Field | Description | Example Value |
|-------|-------------|---------------|
| object-storage-secret | Secret containing S3 access key and secret key, see [Object Storage Credentials](./02_harbor_credential.md#object-storage-credentials) for details | `object-storage-secret` |
| bucket | Name of the object storage bucket | `harbor-registry` |
| regionendpoint | Endpoint URL of the object storage service (include port if needed) | `http://192.168.133.37:32227` |
| region | Region of the object storage (typically `us-east-1` for MinIO) | `us-east-1` |


:::info
Harbor currently only supports configuring the Registry component to use S3 storage. Other components will continue to use PVC or StorageClass for persistent storage.

:::

#### Network Access (YAML Snippets)

Network access mainly includes two methods: domain name access and NodePort access.

Domain name access configuration snippet:

```yaml
spec:
  helmValues:
    expose:
      type: ingress
      tls:
        enabled: false
      ingress:
        hosts:
          core: <domain name>

    externalURL: http://<domain name>
```

NodePort access configuration snippet:

```yaml
spec:
  helmValues:
    expose:
      type: nodePort
      nodePort:
        name: harbor
        ports:
          http:
            port: 80
            nodePort: <port number>

    externalURL: http://<node IP>:<port number>
```

##### Redis Access Credential Configuration \{#configure-redis-credentials}

This refers to the configuration snippet for these credentials in the Harbor instance after configuring the Redis credentials `secret` resource:

Standalone example:

```yaml
spec:
  helmValues:
    database:
    redis:
      external:
        addr: "<redis access address>:<redis port>"
        existingSecret: <secret storing redis password>
        existingSecretKey: password
      type: external
```

Sentinel example:

```yaml
spec:
  helmValues:
    database:
    redis:
      external:
        addr: "<sentinel1 access address>:<sentinel1 port>,<sentinel2 access address>:<sentinel2 port>,<sentinel3 access address>:<sentinel3 port>"
        sentinelMasterSet: mymaster
        existingSecret: <secret storing redis password>
        existingSecretKey: password
      type: external
```

##### PostgreSQL Access Credential Configuration \{#configure-postgresql-credentials}

This refers to the configuration snippet for these credentials in the Harbor instance after configuring the PostgreSQL credentials `secret` resource:

```yaml
spec:
  helmValues:
    database:
      external:
        host: <postgresql access address>
        port: <postgresql port>
        sslmode: <whether to enable ssl>
        username: <postgresql username>
        coreDatabase: <database name>
        existingSecret: <secret storing postgresql password>
      type: external
```

##### Admin Account Configuration

This refers to the configuration snippet for these credentials in the Harbor instance after configuring the account credentials `secret` resource:

```yaml
spec:
  helmValues:
    existingSecretAdminPassword: <secret storing harbor admin password>
    existingSecretAdminPasswordKey: password
```

#### Complete YAML Example: Single Instance, Node Storage, NodePort Network Access

```yaml

spec:
  helmValues:
    existingSecretAdminPassword: harbor-password # Secret storing harbor admin account password
    existingSecretAdminPasswordKey: password # Secret key storing harbor admin account password
    externalURL: http://192.168.142.11:32001 # Harbor access address
    expose:
      nodePort:
        name: harbor
        ports:
          http:
            nodePort: 32001 # Harbor node port number
            port: 80
      type: nodePort
    persistence:
      enabled: true
      hostPath:
        jobservice:
          path: /data/harbor/jobservice # Jobservice component node storage path
        registry:
          path: /data/harbor/registry # Registry component node storage path
        trivy:
          path: /data/harbor/trivy # Trivy component node storage path
    portal:
      resources:
        request:
          cpu: 100m
          memory: 128Mi
        limit:
          cpu: 200m
          memory: 256Mi
    nginx:
      resources:
        request:
          cpu: 100m
          memory: 128Mi
        limit:
          cpu: 200m
          memory: 256Mi
    core:
      resources:
        request:
          cpu: 200m
          memory: 256Mi
        limit:
          cpu: 400m
          memory: 512Mi
    registry:
      nodeSelector:
        kubernetes.io/hostname: 192.168.142.11 # Registry component node selector
      resources:
        request:
          cpu: 200m
          memory: 512Mi
        limit:
          cpu: 400m
          memory: 1Gi
    jobservice:
      nodeSelector:
        kubernetes.io/hostname: 192.168.142.11 # Jobservice component node selector
      resources:
        request:
          cpu: 200m
          memory: 256Mi
        limit:
          cpu: 400m
          memory: 512Mi
    trivy:
      skipUpdate: true
      offlineScan: true
      nodeSelector:
        kubernetes.io/hostname: 192.168.142.11 # Trivy component node selector
      resources:
        request:
          cpu: 200m
          memory: 512Mi
        limit:
          cpu: 400m
          memory: 1Gi
    database:
      external:
        host: harbor-database # Database access address
        port: 5432 # Database port number
        sslmode: require # Whether to enable SSL
        username: harbor # Database username
        coreDatabase: registry # Database name
        existingSecret: harbor-database # Secret storing database password
        existingSecretKey: password # Secret key storing database password
      type: external
    redis:
      external:
        addr: harbor-redis:6379 # Redis access address and port number
        existingSecret: harbor-redis # Secret storing Redis password
        existingSecretKey: password # Secret key storing Redis password
      type: external
```

#### Complete YAML Example: High Availability, Storage Class, Ingress Network Access

```yaml
spec:
  helmValues:
    existingSecretAdminPassword: harbor-password # Secret storing harbor admin account password
    existingSecretAdminPasswordKey: password # Secret key storing harbor admin account password
    externalURL: http://harbor.example.com # Harbor access address
    expose:
      type: ingress
      tls:
        enabled: false
      ingress:
        hosts:
          core: harbor.example.com # Harbor domain name
    persistence:
      enabled: true
      persistentVolumeClaim:
        registry:
          storageClass: ceph # Registry component storage class
          accessMode: ReadWriteMany
          size: 10Gi # Registry component storage size
        jobservice:
          jobLog:
            storageClass: ceph # Jobservice component storage class
            accessMode: ReadWriteMany
            size: 1Gi # Jobservice component storage size
        trivy:
          storageClass: ceph # Trivy component storage class
          accessMode: ReadWriteMany
          size: 5Gi # Trivy component storage size
    portal:
      replicas: 2 # Portal component replicas
      resources:
        request:
          cpu: 200m
          memory: 256Mi
        limit:
          cpu: 400m
          memory: 512Mi
    core:
      replicas: 2 # Core component replicas
      resources:
        request:
          cpu: 200m
          memory: 256Mi
        limit:
          cpu: 400m
          memory: 512Mi
    registry:
      replicas: 2 # Registry component replicas
      resources:
        request:
          cpu: 400m
          memory: 1Gi
        limit:
          cpu: 800m
          memory: 2Gi
    jobservice:
      replicas: 2 # Jobservice component replicas
      resources:
        request:
          cpu: 200m
          memory: 256Mi
        limit:
          cpu: 400m
          memory: 512Mi
    trivy:
      skipUpdate: true
      offlineScan: true
      resources:
        request:
          cpu: 400m
          memory: 1Gi
        limit:
          cpu: 800m
          memory: 2Gi
    database:
      external:
        host: harbor-database # Database access address
        port: 5432 # Database port number
        sslmode: require # Whether to enable SSL
        username: harbor # Database username
        coreDatabase: registry # Database name
        existingSecret: harbor-database # Secret storing database password
        existingSecretKey: password # Secret key storing database password
      type: external
    redis:
      external:
        addr: harbor-redis:6379 # Redis access address and port number
        existingSecret: harbor-redis # Secret storing Redis password
        existingSecretKey: password # Secret key storing Redis password
      type: external

```

## <span id ="harbor_day2">Subsequent Operations</span>

### <span id ="harbor_sso">Configuring Single Sign-On (SSO)</span>

Configuring SSO involves the following steps:

1. Register an SSO authentication client in the global cluster
2. Prepare SSO authentication configuration
3. Configure the Harbor instance to use SSO authentication

Create the following OAuth2Client resource in the global cluster to register the SSO authentication client:

```yaml
apiVersion: dex.coreos.com/v1
kind: OAuth2Client
name: OIDC
metadata:
  name: m5uxi3dbmiwwizlyzpzjzzeeeirsk # This value is calculated based on the hash of the id field, online calculator: https://go.dev/play/p/QsoqUohsKok
  namespace: cpaas-system
alignPasswordDB: true
id: harbor-dex # Client id
public: false
redirectURIs:
  - <harbor access address>/c/oidc/callback
secret: Z2l0bGFiLW9mZmljaWFsLTAK # Client secret
spec: {}
```

Edit the Harbor instance to add the following configuration:

```yaml
spec:
  helmValues:
    oidc:
      enable: true
      clientID: "harbor-dex"
      clientSecret: "Z2l0bGFiLW9mZmljaWFsLTAK"
      issuer: "<platform access address>/dex"
```

### <span id ="harbor_https">Configuring HTTPS</span>

After deploying the Harbor instance, HTTPS can be configured as needed.

First, create a TLS certificate secret in the namespace where the instance is located:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-tls-cert
  namespace: harbor
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
```

Then edit the Harbor instance's YAML configuration to enable HTTPS access:

```yaml
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls-cert
  ingress:
    hosts:
      core: <domain name>

externalURL: https://<domain name>
```

### <span id ="harbor_https">Configuring Image Scanning Vulnerability Database Policy</span>
Harbor's image scanning functionality is implemented by the Trivy component. Considering the user's network environment, the default vulnerability database policy for this component is to use the built-in offline vulnerability database. Since the vulnerability database is not updated, it cannot detect new vulnerabilities in a timely manner.

If you want to keep the vulnerability database up-to-date, you can edit the Harbor instance's YAML configuration to enable the online update policy (this policy requires access to GitHub):
```yaml
trivy:
  skipUpdate: false
  offlineScan: false
```
After enabling the online update policy, Trivy will determine whether to update the vulnerability database before scanning based on the last update time. Since downloading the vulnerability database takes some time, if you don't need Java vulnerability scanning, you can also edit the Harbor instance's YAML configuration to disable Java vulnerability database updates:
```yaml
trivy:
  skipJavaDBUpdate: true
```

## Additional Information

### Deploying Harbor in IPv6 Environment

Harbor supports deployment in IPv6 environments, but you need to ensure that your client tools' versions support IPv6. If you encounter an `invalid reference format` error, please check whether your client tool version supports IPv6.

Related community issues:

- https://github.com/distribution/distribution/pull/3489
- https://github.com/containers/buildah/discussions/4928

### Pod Security Policy \{#pod-security-policy}

When deploying Harbor in a namespace with Pod Security Policy (PSP) configured, the deployment support is as follows:

- StorageClass and PVC: Deployment is supported with enforce level set to either `privileged` or `baseline`.
- Local node storage: Deployment is only supported with enforce level set to `privileged`.
