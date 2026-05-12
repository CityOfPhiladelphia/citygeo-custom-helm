# cnpg-cluster

This helm chart is a very thin wrapper around CloudNativePG to deploy a PostgreSQL cluster. The main purpose is to ensure that we have standardized defaults for backups, compression, restoring, etc.

## Structure

This chart essentially passes through your entire config for each object type, while manually setting a couple special parameters to ensure everything works together.

## Configure

### Minimal Required Configuration

While the chart tries to include a plethora of defaults to allow for minimal configuration for a working cluster, some parameters are required to handle secrets, credentials, endpoints, etc.

```yaml
# Must include the CloudNativePG backup IAM role
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "${cnpg_role}"
cluster:
  # Must include the secret for the postgres user
  superuserSecret:
    name: "airflow-postgres"
objectStore:
  # Must include the S3 destination for backup storage
  configuration:
    destinationPath: "s3://${cnpg_backup_s3}"
```

### Recommended Minimal Configuration

While a database can technically be deployed with the minimum required configuration, you most likely want to include a few more options. In this example, all the values are the defaults.

```yaml
# Must include the CloudNativePG backup IAM role
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "${cnpg_role}"
cluster:
  instances: 2
  storage:
    size: 20Gi
  walStorage:
    size: 12Gi
  # Must include the secret for the postgres user
  superuserSecret:
    name: "airflow-postgres"
  priorityClass: app-critical
  # CPU and memory allocation
  resources:
    requests:
      memory: "1024Mi"
      cpu: "100m"
    limits:
      # Memory limits should always equal requests
      memory: "1024Mi"
  postgresql:
    parameters:
      # Set to roughly 1/4 of total memory
      shared_buffers: "256MB"
objectStore:
  retentionPolicy: '10d'
  # Must include the S3 destination for backup storage
  configuration:
    destinationPath: "s3://${cnpg_backup_s3}"
```

### Additional configuration items

Remember that the entirety of the blocks of `cluster`, `objectStore`, `scheduledBackup`, `databases[]` are passed directly through to the actual resource. Therefore, you can include configuration settings that are not included in the default `values.yaml` and they will still be passed on.
