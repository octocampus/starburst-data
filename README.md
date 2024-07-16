# Starburst Enterprise Deployment with Iceberg, Cloudera Data Platform (CDP) version 7.1.9, and MinIO S3

This guide provides comprehensive steps for deploying Starburst Enterprise, configuring CDP with Iceberg table format, and integrating with MinIO S3

# Prerequisites

- Kubernetes cluster
- Helm installed
- Cloudera Data plateform 7.1.9
- MinIO S3 deployed in Kubernetes

# Steps

## Starburst Enterprise Deployment

1.  Deploy Starburst Coordinator and Worker Pods:

- Use Helm charts for deployment:

```sh
helm repo add --username=<username> --password=<password> starburstdata https://harbor.starburstdata.net/chartrepo/starburstdata
```

- Run the following command to update the cache information about the repo you just added:

```sh
helm repo update
```

- Pull the Starburst Enterprise Values file from Harbor:

```sh
  helm pull --version <version> starburstdata/starburst-enterprise
```

- Store your starburstdata.license file in a Kubernetes secret:

```sh
kubectl create secret generic starburstdata --from-file starburstdata.license
```

- Deploy Starburst Enterprise

```sh
helm upgrade starburst starburstdata/starburst-enterprise --install --version <version> --values registry-access.yaml --values error-stop-sep-cluster.yaml
```

for more information you can see this link: https://docs.starburst.io/latest/k8s.html

```sh
https://docs.starburst.io/latest/k8s.html
```

## Configure CDP 1.7.9 with Iceberg Table Format

1.  CDP - Open Datalakehouse
    Open Data Lakehouse components:
    - Support for Apache Iceberg 1.3 access and processing in CDP Private Cloud Base 7.1.9
    - Compute engines (Impala, Spark, Flink, Nifi) integration for accessing and processing Iceberg datasets concurrently
    - SDX integration with Iceberg catalog
    - Iceberg table maintenance from Spark and replication
    - Iceberg Catalog set to HiveCatalog for Metastore management of Iceberg Tables
    - Certified HDFS and Ozone storage

## Note:

CDP Open Data Lakehouse does not support queries of Iceberg tables from the Hive compute engine in this release.

If the Iceberg storage handler is not included in Hive's classpath, Hive won't be able to manage the metadata for an Iceberg table with the storage handler set. To prevent issues in Hive, Iceberg doesn't add the storage handler to a table unless Hive support is activated. The storage handler is synchronized (added or removed) whenever Hive support is toggled in the table properties. You can enable Hive support in two ways: globally in Hadoop Configuration or on a per-table basis using a table property.

## Configure Starburst Entreprise with Cloudera Data Platform (CDP).

1.  Cloudera Data Platform support (Use the Starburst Hive connector to query Cloudera Data Platform (CDP) version 7.1 or higher.)
2.  Configuration
    - Edit your catalog properties file using the Hive connector
    - Set the metastore to use thrift-cdp7
    - Configure the URI to point to CDP's own Hive Metastore (HMS) Thrift service

```sh
connector.name=hive
hive.metastore=thrift-cdp7
hive.metastore.uri=thrift://cdp-master:9083
```

3.  HDFS file system support
    Trino includes support to access the Hadoop Distributed File System (HDFS) with a catalog using the Delta Lake, Hive, Hudi, or Iceberg connectors.Apache Hadoop HDFS 2.x and 3.x are supported
4.  Configure Starburst Iceberg connector with HDFS
    Requirements
    - Fulfill the Iceberg connector requirements
    - Hive Metastore 3.1.2 or later

### Performance

- Dynamic filtering, and specifically also dynamic row filtering, is enabled by default. Row filtering improves the effectiveness of dynamic filtering for a connector by using dynamic filters to remove unnecessary rows during a table scan. It is especially powerful for selective filters on columns that are not used for partitioning, bucketing, or when the values do not appear in any clustered order naturally.

### Security

- If you have enabled built-in access control for SEP, you must add the following configuration to all Iceberg catalogs:

```sh
iceberg.security=system
```

### Limitations

- Presorting tables is not supported.

## Configure Starburst Enterprise with Iceberg MinIO S3

1. Install the Deployment of Hive:
   The Hive Metastore Service (HMS) is required for the hive connector.

```sh
helm upgrade hive starburstdata/starburst-hive --install --version <version> --values registry-access.yaml --values base-hive.yaml
```