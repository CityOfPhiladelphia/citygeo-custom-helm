# cnpg-cluster

This helm chart is a very thin wrapper around CloudNativePG to deploy a PostgreSQL cluster. The main purpose is to ensure that we have standardized defaults for backups, compression, restoring, etc.

## Structure

This chart essentially passes through your entire config for each object type, while manually setting a couple special parameters to ensure everything works together.

## Minimal required configuration

While the chart tries to include a plethora of defaults to allow for minimal configuration for a working cluster, some parameters are required to handle secrets, credentials, endpoints, etc.

```yaml
values:
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
