# Starburst Enterprise / Dremio Entreprise Deployment with Iceberg, Cloudera Data Platform (CDP) version 7.1.9, and MinIO S3

This guide provides comprehensive steps for deploying Starburst Enterprise on kubernetes cluster, configuring CDP with Iceberg table format, and integrating with MinIO S3

# Global achitecture

<img width="1687" alt="Screenshot 2024-08-29 at 12 24 30" src="https://github.com/user-attachments/assets/397c1b8d-f18d-4b6c-b792-104af094160d">

# Prerequisites

- Kubernetes cluster
- Helm installed
- Cloudera Data plateform 7.1.9
- MinIO S3 deployed in Kubernetes

# Steps

## Starburst Enterprise Deployment

1.  Deploy Starburst Coordinator and Worker Pods:

- Use Helm charts for deployment:
  Don’t forget to replace <version> with the version of Starburst Enterprise you want to deploy.(I used version 448.0.0)

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

### Configure the query logger
  
   The Starburst Enterprise platform (SEP) query logger is the backend service that stores information for:
     - Query completion details and events
     - Cluster metrics
     - Data products
     - Built-in role-based access control
     - Managed statistics
   You must provide and configure a suitable database, and enable the service as described in the requirements and installation sections that follow.
   Requirements:

     - MySQL 8.0.12+
     - PostgreSQL 9.6+
     - OracleDB 12.2.0.1+
     - Network access from the coordinator to the external database.
       
   postgresqlVersion=12.0 is used as the backend service.
       
 ### Configure catalogs
   
     - Connector vs Catalog
       In Starburst, you configure a catalog, which has many properties. One of those properties is the connector. You can think of the connector as the star of the show in the catalog because each catalog can only have one        connector, and the rest of the properties are all specific to that connector. However, you can have multiple catalogs that each contain the same connector. For instance, suppose you have two Teradata Systems. In this 
       case, you would have a separate catalog for each, but both of those catalogs would have the same type of connector.
       You might also have more than one catalog pointing to the same data source, but with different properties for each catalog. Perhaps you want to set authorization differently for different groups of users. Whatever 
       setup you choose, the important thing to remember is connectors are the key property in the catalog configuration.
       

### Upgrade the starburst deployment

  ```sh
helm upgrade starburst starburstdata/starburst-enterprise --install --version 448.0.0 --namespace spark --values registry-access.yaml --values auth-password-biac-sep-prod-cluster-setup.yaml --values catalog-minio-s3.yaml --values catalog-hive-iceberg.yaml --values catalog-minio-ssl.yaml
  ```

### For more information visit this link:

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

### Note:

CDP Open Data Lakehouse does not support queries of Iceberg tables from the Hive compute engine in this release(CDP 7.1.9).

You can check this information in the Cloudera documentation.
```sh
https://docs.cloudera.com/cdp-private-cloud-base/7.1.9/lakehouse-overview/topics/private-cloud-open-data-lakehouse.html
```
If the Iceberg storage handler is not included in Hive's classpath, Hive won't be able to manage the metadata for an Iceberg table with the storage handler set. To prevent issues in Hive, Iceberg doesn't add the storage handler to a table unless Hive support is activated. The storage handler is synchronized (added or removed) whenever Hive support is toggled in the table properties. You can enable Hive support in two ways: globally in Hadoop Configuration or on a per-table basis using a table property.

### Configure Starburst Entreprise with Cloudera Data Platform (CDP).

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

#### Performance

- Dynamic filtering, and specifically also dynamic row filtering, is enabled by default. Row filtering improves the effectiveness of dynamic filtering for a connector by using dynamic filters to remove unnecessary rows during a table scan. It is especially powerful for selective filters on columns that are not used for partitioning, bucketing, or when the values do not appear in any clustered order naturally.

#### Security

- If you have enabled built-in access control for SEP, you must add the following configuration to all Iceberg catalogs:

```sh
iceberg.security=system
```

#### Limitations

- Presorting tables is not supported.

## Configure Starburst Enterprise/ Dremio Entreprise with Iceberg MinIO S3

### Install the Deployment of Hive

   The Hive Metastore Service (HMS) is required for the hive connector. 
   Run the following command in a terminal.

```sh
helm upgrade hive starburstdata/starburst-hive --install --version <version> --values registry-access.yaml --values base-hive.yaml
```
### Configure SEP catalogs to connect with MinIO S3 using Apache Iceberg as the table format and a Hive Metastore deployed in Kubernetes

```sh
    connector.name=iceberg
    iceberg.catalog.type=hive_metastore
    iceberg.security=system
    hive.metastore.uri=thrift://hive.spark:9083
    hive.s3.aws-access-key='Your accces key from minio'
    hive.s3.aws-secret-key='Your secret Key from minio'
    hive.s3.endpoint=''
    hive.s3.path-style-access=true
```
### Configure Dremio entreprise to connect with MinIO S3 using Apache Iceberg as the table format and a Hive Metastore deployed in Kubernetes

Adding a new data source in Dremio is a straightforward process that can be done directly from the user interface.
Click on the "Add Source" button, A list of available source types will appear. Dremio supports a wide range of sources, including HDFS, Hive, S3, and various SQL databases like MySQL and PostgreSQL. 
Select Hive Hive 3.x .

Here's the configuration:

<img width="1212" alt="Screenshot 2024-08-28 at 23 21 40" src="https://github.com/user-attachments/assets/d8e50c79-01a8-4f2c-8b2e-35f16b02eea2">



<img width="1205" alt="Screenshot 2024-08-28 at 23 21 29" src="https://github.com/user-attachments/assets/be919019-97db-44be-a697-4d1f6cc426a6">

### Summary

Dremio and Starburst Enterprise are both configured to query the same Apache Iceberg table stored in MinIO S3, utilizing a shared Hive Metastore deployed in a Kubernetes environment. This setup allows both platforms to access the same data source, enabling consistent data management and analysis across different query engines


# Benshmarking Straburst Entreprise & Dremio Entreprise using Jmeter

### Steps to Install Apache JMeter
1. Download Apache JMeter (I installed apache-jmeter-5.6.3 2 and once the download is complete, extract the JMeter archive to a location of your choice)
2. Install Java (if not already installed). JMeter requires Java (JDK or JRE) to be installed. Ensure you have at least Java 8 or a newer version.
3. Start Apache JMeter
   - Navigate to the JMeter bin directory
   - Run the JMeter GUI application with this command:
     
   ```sh
   sh jmeter.sh
   ```
The JMeter GUI will open, and you can start creating your test plans

### Note : Java Compatibility for Dremio Enterprise

  If you are using Java 9 or a newer version, certain internal Java modules must be exposed to avoid compatibility issues with Apache Arrow. This is done by adding specific --add-opens flags to your Java command.
  
  Here’s how to add the necessary flag
  Directly on the command line:

   ```sh
   vim jmeter.sh
   ```

   ```sh
  #--add-opens if JAVA 9
  JAVA9_OPTS="--add-opens java.desktop/sun.awt=ALL-UNNAMED \
  --add-opens java.desktop/sun.swing=ALL-UNNAMED \
  --add-opens java.desktop/javax.swing.text.html=ALL-UNNAMED \
  --add-opens java.desktop/java.awt=ALL-UNNAMED \
  --add-opens java.desktop/java.awt.font=ALL-UNNAMED \
  --add-opens java.base/java.lang=ALL-UNNAMED \
  --add-opens java.base/java.lang.invoke=ALL-UNNAMED \
  --add-opens java.base/java.lang.reflect=ALL-UNNAMED \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/java.text=ALL-UNNAMED \
  --add-opens java.desktop/sun.awt.X11=ALL-UNNAMED \
  --add-opens java.desktop/sun.awt.shell=ALL-UNNAMED \
  --add-opens java.base/java.nio=ALL-UNNAMED \
  --add-opens java.base/java.io=ALL-UNNAMED \
  --add-opens java.base/java.nio=org.apache.arrow.memory.core,ALL-UNNAMED"
  ```

4. Create a New Test Plan
   - create one manually by clicking on File > New
5. Add a Thread Group
   - Right-click on the Test Plan > Add > Threads (Users) > Thread Group
   - The Thread Group is where you define the number of users (threads), ramp-up period (time to start all users), and loop count (number of times to execute the test.
     
6. Add an JDBC Connection configuration
   
 - Right-click on Thread Group > Config Elemnt > JDBC Connection configuration
 - Right-click on Test plan > Add > Sampler > HTTP Request.

Here's how you can configure connection from Jmeter to Starburst Entreprise

<img width="1727" alt="Screenshot 2024-08-29 at 11 11 03" src="https://github.com/user-attachments/assets/5cd891ed-8eb0-4e12-8af5-03ee2bb0c595">

Here's how you can configure connection from Jmeter to Dremio Entreprise

<img width="1728" alt="Screenshot 2024-08-29 at 11 12 51" src="https://github.com/user-attachments/assets/67df0e2b-c6ee-4972-9f06-3f7ac3f8a221">

7. Add a Listener to View Results
   
 - Right-click on Thread Group > Add > Listener > View Results Tree.
 - Right-click on Thread Group > Add > Listener > Summary Report.

### Summary

JMeter was utilized to benchmark the query performance of Starburst Enterprise and Dremio when accessing the same Iceberg tables stored in MinIO S3, using a Hive Metastore deployed in Kubernetes.These results can guide optimization efforts and inform decisions on choosing the right query engine based on specific use cases and workload requirements




   
   










