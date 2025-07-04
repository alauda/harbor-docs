---
title: "Configuring Redis, PostgreSQL, and Account Access Credentials"
description: ""
weight: 100
---

This document describes the configuration methods for credentials required by Harbor instances.

## Prerequisites

- This document applies to Harbor 2.12 and above versions provided by the platform. It is decoupled from the platform based on technologies such as Operator.

## Redis Credentials \{#redis-credentials}

### Requirements

Harbor has the following requirements for Redis deployment:

- Redis version 7.x is recommended.
- Deployment modes support both `Standalone` and `Sentinel` modes. However, Redis `Cluster` mode is not supported.

For detailed Redis deployment instructions, please refer to the [Harbor Official Documentation](https://goharbor.io/docs/2.12.0/install-config/).

### Credential Format

Create a Secret in the namespace where the Harbor instance is planned to be deployed, select the Opaque type, and add and fill in the following fields in the configuration:

| Field                | Description                                                                                 | Arch                | Example Value                                           |
| -------------------- | ------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------- |
| **host**             | Redis connection address. Ensure that the Harbor service can connect to it.                 | standalone          | `192.168.1.1`                                           |
| **port**             | Redis connection port. Ensure that the Harbor service can connect to this port.             | standalone          | `6379`                                                  |
| **password**         | Redis instance account password. Required when Redis authentication is enabled.             | standalone,sentinel | `password111`                                           |
| **address**          | Sentinel node connection address.                                                           | sentinel            | `192.168.1.1:26379,192.168.1.2:26379,192.168.1.3:26379` |
| **masterName**       | The name of the instance group monitored by Sentinel in the sentinel.conf.                  | sentinel            | `mymaster`                                              |

:::warning

1. When both sentinel and standalone configurations are present, the sentinel configuration will take precedence.
2. When deploying with high availability templates, if standalone Redis is configured, it is the user's responsibility to ensure the high availability of the Redis instance.

:::

**Standalone example:**

```yaml
apiVersion: v1
data:
  host: <base64 encode host>
  port: <base64 encode port>
  password: <base64 encode password>
kind: Secret
metadata:
  name: harbor-redis
  namespace: <ns-of-harbor-instance>
type: Opaque
```

**Sentinel example:**

```yaml
apiVersion: v1
data:
  password: <base64 encode password>
  address: <base64 encode address>
  masterName: <base64 encode masterName>
kind: Secret
metadata:
  name: harbor-redis
  namespace: <ns-of-harbor-instance>
type: Opaque
```

### Updating Credentials
If you want to modify Redis connection information after deploying a Harbor instance, you need to directly update the Harbor instance resource, rather than modifying the credential content. For specific operations, please refer to [Configuring Redis Access Credentials](./03_harbor_deploy.md#configure-redis-credentials).

### Using Alauda Cache Service for Redis OSS
When providing Redis service through Alauda Cache Service for Redis OSS, consider the following important requirements:
- Redis version 7.x is recommended.
- Select Sentinel mode for architecture type.
- Choose an RDB persistence template for the parameter template, such as system-rdb-redis-7.2-sentinel.
- Enable data persistence with a storage quota of not less than 2G.
- In multi-network card scenarios, Redis Sentinel selects the default IP of the node to initialize the access address of each Redis node. It does not support accessing nodes with non-default IP + exposed port. Use the LoadBalancer access method to create Redis instances. For more details, refer to the Alauda Cache Service for Redis OSS feature description documentation.


:::warning

When creating a Redis instance, a Secret containing connection information is automatically generated, which can be used directly to deploy Harbor. This Secret resource can be filtered using the label `middleware.instance/type: Redis`.

```bash
kubectl get secret -n <ns-of-redis-instance> -l middleware.instance/type=Redis
```

If the Redis instance and Harbor instance are not in the same namespace, you need to copy the Secret resource to the namespace where the Harbor instance is located.

For more Redis deployment parameters and high availability deployment requirements, please refer to the <ExternalSiteLink name="redis" href="/installation.html" children="Redis Deployment Documentation" />.

:::


## PostgreSQL Credentials \{#pg-credentials}

### Requirements

Harbor has the following requirements for PostgreSQL versions:

- Harbor 2.12.x requires PostgreSQL version 14.x

### Credential Format

Create a Secret in the namespace where the Harbor instance is planned to be deployed, select the Opaque type, and add and fill in the following fields in the configuration:

| Field         | Description                                                                                                | Example Value   |
|---------------|------------------------------------------------------------------------------------------------------------|----------------|
| **host**      | Database connection address. Ensure that the Harbor service can connect to this database address.           | `192.168.1.1`  |
| **port**      | Database connection port. Ensure that the Harbor service can connect to this database port.                 | `5432`         |
| **username**  | Database account username                                                                                   | `postgres`       |
| **password**  | Database account password                                                                                   | `password111`  |
| **database**  | Database name. This database must already exist and be empty. You can use the command `create database <database name>` to create a database | `harbor_db`    |
| **sslmode**   | Whether to enable SSL for database connections. Available options:<br>- `require`: Require SSL connection<br>- `disable`: Disable SSL connection <br>- `verify-ca`: Verify the server's certificate<br>- `verify-full`: Verify the server's certificate and hostname. more about [sslmode](#sslmode) | `require`       |

*YAML* example:

```yaml
apiVersion: v1
data:
  database: <base64 encode database name>
  host: <base64 encode host>
  password: <base64 encode password>
  port: <base64 encode port>
  username: <base64 encode username>
  sslmode: <base64 encode sslmode>
kind: Secret
metadata:
  name: harbor-database
  namespace: <ns-of-harbor-instance>
type: Opaque
```

**How to Create a Database on a PG Instance**

Connect to the PG instance using the psql cli and execute the following command to create a database:

```bash
create database <database name>;
```

### sslmode \{#sslmode}

sslmode is a parameter that controls the security of the connection between the Harbor service and the PostgreSQL database. Available options:

- `require`: Require SSL connection
- `disable`: Disable SSL connection
- `verify-ca`: Verify the server's certificate
- `verify-full`: Verify the server's certificate and hostname


When you use `Alauda support for PostgreSQL`, the `sslmode` should be set to `require`. <br>
When you use external PostgreSQL, the `sslmode` is depends on your PostgreSQL configuration.

### Updating Credentials

If you want to modify PostgreSQL connection information after deploying a Harbor instance, you need to directly update the Harbor instance resource, rather than modifying the credential content. For specific operations, please refer to [Configure PostgreSQL Credentials](./03_harbor_deploy.md#configure-postgresql-credentials).

### Using PostgreSQL Provided by Data Services

`Data Services` supports deploying PostgreSQL instances that can be used for Harbor deployment. When creating a PostgreSQL instance, please consider the following important requirements:

1. Choose a PostgreSQL version that matches your Harbor version, for example, when deploying Harbor 2.12.x, you need to select PostgreSQL 14.x
2. Storage quota should not be less than 5Gi

When creating a PostgreSQL instance, a Secret containing connection information is automatically generated. This Secret resource can be filtered using the label `middleware.instance/type: PostgreSQL`.

```bash
kubectl get secret -n <ns-of-postgresql-instance> -l middleware.instance/type=PostgreSQL | grep -E '^postgres'
```

This Secret contains `host`, `port`, `username`, `password` information. You need to supplement `database` and `sslmode` (set to `require`) information based on this Secret, and create a new secret in the namespace where the Harbor instance is located.

When creating a Postgres instance, a Secret that starts with postgres and contains connection information is automatically generated. This Secret can be directly utilized for Harbor deployment and can be filtered using the following command:

```bash
kubectl get secret -n <ns-of-postgres-instance> -l middleware.instance/type=PostgreSQL
```

If the Postgres instance and Harbor instance are not in the same namespace, you need to copy the Secret resource to the namespace where the Harbor instance is located.

For more PostgreSQL deployment parameters and requirements, please refer to <ExternalSiteLink name="postgresql" href="/installation.html" children="PostgreSQL Deployment Documentation" />.


## Harbor Account Credentials \{#harbor-credentials}

Create a Secret in the namespace where the Harbor instance is planned to be deployed, select the Opaque type, and add and fill in the following fields in the configuration:

| Field         | Description                                                                                                                                                                                                                                                      | Example Value  |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| **password**  | Set the password for the default admin account, which must contain letters, numbers, and special characters, be at least 8 characters long) cannot be used | `password111@` |
| **namespace** | Set the same namespace as the Harbor instance                                                                                                                                                                                                                    | `tools`        |

Note that the default username for Harbor is admin.

```yaml
apiVersion: v1
data:
  password: <base64 encode password>
kind: Secret
metadata:
  name: harbor-admin-password
  namespace: <ns-of-harbor-instance>
type: Opaque
```
