# Starburst Enterprise & Dremio Entreprise Deployment with Iceberg, Cloudera Data Platform (CDP) version 7.1.9, and MinIO S3

This guide provides comprehensive steps for deploying Starburst Enterprise on kubernetes cluster, configuring CDP with Iceberg table format, and integrating with MinIO S3

# High-Level Achitecture

<img width="1584" alt="Screenshot 2024-09-08 at 19 08 59" src="https://github.com/user-attachments/assets/3133807f-79d4-477c-b494-d56d89fe10b1">


# Prerequisites

- Kubernetes cluster
- Helm installed
- Cloudera Data plateform 7.1.9
- MinIO S3 deployed in Kubernetes

## Table of Contents

- [Starburst Entreprise](#SEP)
    - [What's Straburst Entreprise?](#what-is-Straburst)
    - [Configuring Starburst Enterprise in Kubernetes](#Configuring-SEP-on-k8s)

- [Cloudera Data Platform (CDP) Private Cloud Base7.1.9](#CDP)
   - [Configuring CDP 1.7.9 with Iceberg Table Format](#Configuring-CDP-Iceberg)
   - [Configuring Starburst Entreprise with Cloudera Data Platform (CDP)](#Configuring-Starburst-with-CDP)

- [Configuring Starburst Enterprise & Dremio Entreprise with Apache Iceberg Table Format on MinIO S3](#SEP-DREMIO-ICEBERG-S3)
   - [Install the Deployment of Hive Metastore (HMS)](#HMS)
   - [Configuring SEP catalogs to connect with MinIO S3 using Apache Iceberg as the table format and a Hive Metastore deployed in Kubernetes](#SEP-HMS-on-S3)
   - [Configuring Dremio entreprise to connect with MinIO S3 using Apache Iceberg as the table format and a Hive Metastore deployed in Kubernetes](#)
   - [Summary](#Summary)
     
- [Benshmarking Straburst Entreprise & Dremio Entreprise using Jmeter](#Benshmarking-Straburst-Entreprise-&-Dremio-Entreprise-using-Jmeter)

# What's Straburst Entreprise?

Starburst Enterprise is a high-performance SQL query engine designed for fast and efficient data analytics across various data sources. Built on top of the open-source Trino project, which offers enhanced features for scalability, security, and performance, making it an ideal solution for modern data architectures.

- massively parallel processing (MPP):
  which allows it to handle large-scale queries across distributed systems efficiently, providing faster insights from complex data.
  
- Data federation capabilities:
  Starburst Enterprise enables organizations to query data across multiple sources—such as Hadoop, S3, SQL databases, and cloud storage—without needing to move or replicate the 
  data. This reduces data silos and simplifies data access, ensuring real-time analytics. As a result, businesses can scale their data analytics effortlessly while maintaining 
  high performance, flexibility, and seamless integration across diverse data environments.
  
- SEP ensures that each connector can fully utilize the MPP architecture:
  esulting in improved query performance, lower latency, and better scalability. This level of optimization is crucial for enterprises that require high-performance analytics 
  across diverse and distributed data sources.
  
- RBAC in Starburst Enterprise:
  Starburst Enterprise includes a comprehensive implementation of RBAC that allows organizations to define and manage access controls for users and roles more effectively:
  1. Fine-Grained Access Control:
     allowing administrators to define specific access permissions at various levels, including catalogs, schemas, tables, columns, and even rows. This allows for precise control over who can access or modify specific pieces of data.
  2. Role-Based Permissions:
     Administrators can create roles and assign them specific permissions, making it easy to manage access for groups of users rather than configuring permissions for each  individual user. This supports the principle of least privilege and helps ensure that users only have access to the data they need.
     
     Roles can be hierarchical, inheriting permissions from other roles, which simplifies administration and maintenance.
  4. Integration with Identity Providers:
     Starburst Enterprise integrates with enterprise identity providers like Okta, Azure Active Directory (AD), LDAP, and Kerberos for seamless user authentication and authorization management. This allows organizations to leverage their existing identity management solutions and streamline user access.

     Single Sign-On (SSO) support and Multi-Factor Authentication (MFA) can also be enforced via these identity providers, adding another layer of security.
  5. Built-in access control masks and filters:
     You can use Starburst Enterprise platform (SEP)’s built-in access control to restrict what data users can see at the row and column level. By masking data and filtering rows, you can allow different users to run the same queries, but be prevented from viewing sensitive data. These data obfuscation features are applied to roles using SEP’s ‘built-in     access control privileges.
  6. Auditing and Compliance:
     Starburst Enterprise includes enhanced auditing capabilities that log access and query activities. This is crucial for compliance with data governance policies and regulations like GDPR, HIPAA, and CCPA.
     The audit logs can be integrated with external logging and monitoring systems, such as Splunk or ELK, for better analysis and compliance reporting.
  7. Integration with Apache Ranger:
     Starburst Enterprise can integrate with Apache Ranger to provide centralized policy management for access control across multiple services. This allows organizations to define and enforce consistent security policies across the entire data ecosystem.

## Configuring Starburst Enterprise in Kubernetes

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

### Configuring the query logger
  
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
   In Starburst, you configure a catalog, which has many properties. One of those properties is the connector. You can think of the connector as the star of the show in the 
   catalog because each catalog can only have one connector, and the rest of the properties are all specific to that connector. However, you can have multiple catalogs that each 
   contain the same connector. For instance, suppose you have two Teradata Systems. In this case, you would have a separate catalog for each, but both of those catalogs would have 
   the same type of connector.
   You might also have more than one catalog pointing to the same data source, but with different properties for each catalog. Perhaps you want to set authorization differently 
   for different groups of users. Whatever setup you choose, the important thing to remember is connectors are the key property in the catalog configuration.
       
### Upgrade the starburst deployment

  ```sh
helm upgrade starburst starburstdata/starburst-enterprise --install --version 448.0.0 --namespace spark --values registry-access.yaml --values auth-password-biac-sep-prod-cluster-setup.yaml --values catalog-minio-s3.yaml --values catalog-hive-iceberg.yaml --values catalog-minio-ssl.yaml
  ```

### For more information visit this link:

```sh
https://docs.starburst.io/latest/k8s.html
```

## Configuring CDP 1.7.9 with Iceberg Table Format

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

### Configure Starburst Entreprise with Cloudera Data Platform (CDP)

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
      
### Some features of configuring Apache iceberg with HDFS
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




   
   










