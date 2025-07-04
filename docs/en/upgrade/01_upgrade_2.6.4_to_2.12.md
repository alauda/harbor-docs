# Harbor Migration Guide: From 2.6.4 to 2.12

## Migration Instructions

This guide describes how to upgrade Harbor from version 2.6.4 to version 2.12. Considering the large version gap and upgrade stability, we adopt a data migration approach for the upgrade. The advantages of this approach are:

1. Avoiding the complexity of multiple intermediate version upgrades
2. Reusing the existing database, Redis, and storage with minimal changes.

## I. Data Backup

### Database Backup

```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>

kubectl -n ${INSTANCE_NAMESPACE} exec -it ${INSTANCE_NAME}-database-0 -- bash

# Execute backup command
pg_dump -U postgres -d registry > /tmp/harbor_database.dump

# Copy backup to local machine
kubectl -n ${INSTANCE_NAMESPACE} cp ${INSTANCE_NAME}-database-0:/tmp/harbor_database.dump ./harbor_database.dump
```

### Registry Backup

Since Registry data is typically large, it's not recommended to copy it from the Pod. You need to locate the storage directory based on the corresponding storage type.

You can get the Registry configuration from the harbor instance yaml as follows:
```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>

helm -n ${INSTANCE_NAMESPACE} get values ${INSTANCE_NAME}
```

**Host Path Storage**

If the original instance is deployed with Host Path storage, the configuration is as follows:

```yaml
persistence:
  hostPath:
    registry:
      host:
        nodeName: <node name>
        path: <registry storage path>
```
Enter the directory on the node to backup the data.

**PVC or Storage Class**

If the original instance is deployed with a storage class, the PVC has a fixed name: `<instance name>-harbor-registry`.

If the original instance is deployed with PVC storage, the configuration is as follows:
```yaml
helmValues:
  persistence:
    persistentVolumeClaim:
      registry:
        existingClaim: <registry pvc name>
```

You can view the specific information of the PVC with the following command:

```yaml
export INSTANCE_NAMESPACE=<harbor instance namespace>

kubectl -n ${INSTANCE_NAMESPACE} get pvc <registry pvc name> -o yaml
```
You need to refer to the documentation of the PVC storage class to locate the data directory.


## II. Stop Old Harbor Service

Firstly, uninstall the harbor operator from the Operator Hub page, then scale down the old harbor service by following commands:

```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>

kubectl -n ${INSTANCE_NAMESPACE} scale deployment -l release=${INSTANCE_NAME} --replicas=0
kubectl -n ${INSTANCE_NAMESPACE} scale statefulset -l release=${INSTANCE_NAME} --replicas=0
```

> Note: Do not stop the database component, as we need to perform migration on it

## III. Database Migration

First, get the instance's yaml configuration with the following command:

```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>

helm -n ${INSTANCE_NAMESPACE} get values ${INSTANCE_NAME}
```

The database configuration is as follows:

```yaml
database:
  type: external
  external:
    existingSecret: <database password secret>
    host: <database host>
    port: <database port>
    username: <database username>
```

Then create a job to execute database migration

```bash
export POSTGRESQL_HOST=<database host> POSTGRESQL_PORT=<database port> POSTGRESQL_USERNAME=<database username> POSTGRESQL_PASSWORD=<database password> POSTGRESQL_SSLMODE="require"
export INSTANCE_NAMESPACE=<harbor instance namespace>

kubectl -n ${INSTANCE_NAMESPACE}  apply -f - << EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: harbor-db-migrate
spec:
  backoffLimit: 5
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: harbor-migrate
        image: docker-mirrors.alauda.cn/mgle/standalone-db-migrator:v2.12.0
        command: ["/harbor/migrate"]
        env:
        - name: POSTGRESQL_HOST
          value: "${POSTGRESQL_HOST}"
        - name: POSTGRESQL_PORT
          value: "${POSTGRESQL_PORT}"
        - name: POSTGRESQL_USERNAME
          value: "${POSTGRESQL_USERNAME}"
        - name: POSTGRESQL_PASSWORD
          value: "${POSTGRESQL_PASSWORD}"
        - name: POSTGRESQL_DATABASE
          value: "registry"
        - name: POSTGRESQL_SSLMODE
          value: "${POSTGRESQL_SSLMODE}"
EOF
```

After the job is completed, you can see the specific migration records from the logs

```
2024-12-23T02:43:05Z [INFO] [/cmd/standalone-db-migrator/main.go:53]: Migrating the data to latest schema...
2024-12-23T02:43:05Z [INFO] [/cmd/standalone-db-migrator/main.go:54]: DB info: postgres://harbor@test-6-database:5432/registry?sslmode=require
2024-12-23T02:43:05Z [INFO] [/common/dao/base.go:67]: Registering database: type-PostgreSQL host-test-6-database port-5432 database-registry sslmode-"require"
2024-12-23T02:43:05Z [INFO] [/common/dao/base.go:72]: Register database completed
2024-12-23T02:43:05Z [INFO] [/common/dao/pgsql.go:135]: Upgrading schema for pgsql ...
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 100/u 2.7.0_schema (184.508549ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 110/u 2.8.0_schema (275.019668ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 111/u 2.8.1_schema (286.508359ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 120/u 2.9.0_schema (357.027979ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 130/u 2.10.0_schema (378.349099ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 140/u 2.11.0_schema (399.407785ms)
2024-12-23T02:43:05Z [INFO] [/go/pkg/mod/github.com/golang-migrate/migrate/v4@v4.18.1/migrate.go:765]: 150/u 2.12.0_schema (408.831695ms)
2024-12-23T02:43:05Z [INFO] [/cmd/standalone-db-migrator/main.go:63]: Migration done.  The data schema in DB is now update to date.
```

## IV. Deploy New Harbor Service

> The new instance needs to mount the storage of the original instance, so it needs to be in the same namespace as the original instance.

### Get Original Instance Configuration

We need to get the configuration from the original instance and convert it to the configuration of the new instance, as follows:

```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>

helm -n ${INSTANCE_NAMESPACE} get values ${INSTANCE_NAME}
```

### Convert Database Configuration

```yaml
# Original instance configuration
database:
  type: external
  external:
    existingSecret: <database password secret>
    host: <database host>
    port: <database port>
    sslmode: <whether to enable SSL>
    username: <database username>
```

```yaml
# New instance configuration
helmValues:
  database:
    type: external
    external:
      existingSecret: <database password secret>
      host: <database host>
      port: <database port>
      sslmode: <whether to enable SSL>
      username: <database username>
```

> Note: The database password secret key is different between the old and new instances. The new instance needs to use the POSTGRES_PASSWORD field
```yaml
data:
  POSTGRES_PASSWORD: <password base64>
```

### Convert Redis Configuration

```yaml
# Original instance configuration
redis:
  type: external
  external:
    addr: rfr-harbor-test-5-redis:6379
    existingSecret: test-5-redis-password
```

```yaml
# New instance configuration
helmValues:
  redis:
    type: external
    external:
      addr: <redis host>:<redis port>
      existingSecret: <redis password secret>
```

> Note: The Redis password secret key is different between the old and new instances. The new instance needs to use the REDIS_PASSWORD field
```yaml
data:
  REDIS_PASSWORD: <password base64>
```


### Convert Host Path Storage Configuration

If the original instance is deployed with Host Path storage, the configuration is as follows:

```yaml
# Original instance configuration
persistence:
  hostPath:
    jobservice:
      host:
        nodeName: <node name>
        path: <jobservice storage path>
    registry:
      host:
        nodeName: <node name>
        path: <registry storage path>
    trivy:
      host:
        nodeName: <node name>
        path: <trivy storage path>
```

```yaml
# New instance configuration
helmValues:
  persistence:
    enabled: true
    hostPath:
      registry:
        path: <registry storage path>
      jobservice:
        path: <jobservice storage path>
      trivy:
        path: <trivy storage path>

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

### Convert Storage Class Storage Configuration

If the original instance is deployed with a storage class, the PVC names are fixed:

* jobservce: `<instance name>-harbor-jobservice`
* registry: `<instance name>-harbor-registry`
* trivy: `data-<instance name>-harbor-trivy-0`

```yaml
# New instance configuration
helmValues:
  persistence:
    enabled: true
    persistentVolumeClaim:
      registry:
        existingClaim: <instance name>-harbor-jobservice
      jobservice:
        jobLog:
          existingClaim: <instance name>-harbor-registry
      trivy:
        existingClaim: data-<instance name>-harbor-trivy-0
```

If the original instance is deployed with PVC, the configuration is as follows:

```yaml
# Original instance configuration
persistence:
  persistentVolumeClaim:
    registry:
      existingClaim: <registry pvc name>
    jobservice:
      existingClaim: <jobservice pvc name>
    trivy:
      existingClaim: <trivy pvc name>
```

```yaml
# New instance configuration
helmValues:
  persistence:
    enabled: true
    persistentVolumeClaim:
      registry:
        existingClaim: <registry pvc name>
      jobservice:
        jobLog:
          existingClaim: <jobservice pvc name>
      trivy:
        existingClaim: <trivy pvc name>
```

### Convert NodePort Configuration

If the original instance is deployed with NodePort, the configuration is as follows:

```yaml
# New instance configuration
helmValues:
  expose:
    type: nodePort
    nodePort:
      name: harbor
      ports:
        htt部署
          port: 80
          nodePort: <node port number>

  externalURL: http://<node port ip>:<node port number>
```

### Convert Ingress Configuration

If the original instance is deployed with a domain name, the configuration is as follows:

```yaml
# New instance http configuration
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

```yaml
# New instance https configuration
helmValues:
  expose:
    type: ingress
    tls:
      enabled: true
      certSource: "secret"
      secret:
        secretName: <tls certificate secret>
    ingress:
      hosts:
        core: <domain name>

  externalURL: https://<domain name>
```

## V. Verification

1. Check all Pod statuses:

```bash
export INSTANCE_NAME=<harbor instance name> INSTANCE_NAMESPACE=<harbor instance namespace>
kubectl get pods -n ${INSTANCE_NAMESPACE} -l release=${INSTANCE_NAME}
```

2. Verify Harbor service is accessible:
    * Access Harbor Web UI, verify existing projects and images are visible
    * Test Docker login

3. Test pushing and pulling images
4. After confirming the migration is normal, you can manually delete the old Harbor instance.