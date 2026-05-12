# cnpg-cluster

This helm chart is a very thin wrapper around CloudNativePG to deploy a PostgreSQL cluster. The main purpose is to ensure that we have standardized defaults for backups, compression, restoring, etc.

## Structure

This chart essentially passes through your entire config for each object type, while manually setting a couple special parameters to ensure everything works together.

The chart isn't really that "smart" besides automating connecting the pieces together, and providing defaults that work for our environment.

It does, however, essentially match all features available in CloudNativePG because of the pass-through, so you don't have to worry about whether or not the chart supports a feature.

## Configure

### Minimal Required Configuration

While the chart tries to include a plethora of defaults to allow for minimal configuration for a working cluster, some parameters are required to handle secrets, credentials, endpoints, etc.

```yaml
# Must include the CloudNativePG backup IAM role
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: "${cnpg_role}"
cluster:
  spec:
    # Must include the secret for the postgres user
    superuserSecret:
      name: "example-postgres"
objectStore:
  spec:
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
  spec:
    instances: 2
    storage:
      size: 20Gi
    walStorage:
      size: 12Gi
    # Must include the secret for the postgres user
    superuserSecret:
      name: "example-postgres"
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
  spec:
    retentionPolicy: '10d'
    # Must include the S3 destination for backup storage
    configuration:
      destinationPath: "s3://${cnpg_backup_s3}"
```

### Additional configuration items

Remember that the entirety of the blocks of `cluster`, `objectStore`, `scheduledBackup`, `databases[]` are passed directly through to the actual resource. Therefore, you can include configuration settings that are not included in the default `values.yaml` and they will still be passed on.

## Recover from backup

This helm chart makes recovering from backup very easy, especially when using FluxCD.

1. Copy your entire helm release.
1. Change `metadata.name` and `spec.releaseName` to a new value, perhaps the previous name with `-restore` at the end
1. Add this to the end of `spec.values.cluster.spec` (if you already had a bootstrap section, just add the extra recovery field)

```yaml
bootstrap:
  recovery:
    source: origin
    recoveryTarget:
      # Time base target for the recovery
      # (comment out to restore to latest moment)
      targetTime: "2026-04-27 15:05:21-04:00"
externalClusters:
  - name: origin
    plugin:
      name: barman-cloud.cloudnative-pg.io
      parameters:
        barmanObjectName: # Name of previous release
        serverName: # Name of previous release
```

Deploy the cluster and it will do the rest for you!

### Backup recovery example

Original cluster:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: example-psql
  namespace: flux-system
spec:
  releaseName: example-psql
  targetNamespace: example
  timeout: 10m
  interval: 15m
  chart:
    spec:
      version: 0.x.x
      chart: cnpg-cluster
      sourceRef:
        kind: HelmRepository
        name: citygeo-custom-helm
        namespace: flux-system
  values:
    # Using values from "Minimal Required Configuration" example
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "${cnpg_role}"
    cluster:
      spec:
        superuserSecret:
          name: "airflow-postgres"
    objectStore:
      spec:
        configuration:
          destinationPath: "s3://${cnpg_backup_s3}"
```

Restored  cluster:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  # ! CHANGED
  name: example-psql-restore
  namespace: flux-system
spec:
  # ! CHANGED
  releaseName: example-psql-restore
  targetNamespace: example
  timeout: 10m
  interval: 15m
  chart:
    spec:
      version: 0.x.x
      chart: cnpg-cluster
      sourceRef:
        kind: HelmRepository
        name: citygeo-custom-helm
        namespace: flux-system
  values:
    # Using values from "Minimal Required Configuration" example
    serviceAccount:
      annotations:
        eks.amazonaws.com/role-arn: "${cnpg_role}"
    cluster:
      spec:
        superuserSecret:
          name: "airflow-postgres"
        # ! ADDED RECOVERY PARAMS
        bootstrap:
          recovery:
            source: origin
            recoveryTarget:
              # Time base target for the recovery
              # (comment out to restore to latest moment)
              targetTime: "2026-04-27 15:05:21-04:00"
        externalClusters:
          - name: origin
            plugin:
              name: barman-cloud.cloudnative-pg.io
              parameters:
                barmanObjectName: example-psql # Name of previous release
                serverName: example-psql # Name of previous release
        # ! END OF BACKUP RECOVERY PARAMS
    objectStore:
      spec:
        configuration:
          destinationPath: "s3://${cnpg_backup_s3}"
```
