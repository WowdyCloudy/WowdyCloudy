# Databricks Architecture

Databricks is a **Paas** (Platform-as-a-Service) cloud service offering that is integrated with all three major cloud providers. A **Databricks cloud account** can be opened on Amazon AWS, Microsoft Azure,
or Google Cloud Platform. One or more workspaces can then be deployed into AWS/Azure/GCP regions via a Databricks account. A **workspace** is the primary environment that engineering and analytical teams
use for working with Databricks objects and clusters, it connects their code/queries to compute resources and data assets. Multiple workspaces can be deployed into one or more cloud provider regions.
In a multi-cloud scenario, one Databricks account per provider is required for using workspaces on different platforms.

The following image provides a high-level Databricks architecture overview, the sections below describe its most important entities in more detail:

<img src="../resources/images/dbx-arch.png" width="500" height="500" />

## Platform Planes
Databricks workspaces deployed to Amazon Web Services, Microsoft Azure, and Google Cloud Platform follow similar architectures consisting of two main components/layers:
- An internal **Control Plane** which is hosted in Databricks's cloud account. It stores workspace configurations and contains the core backend functionalities relating to the deployment (Web UI, Cluster administration, Job Scheduler, Rest APIs, ...)
- **Compute** (or Data) **Planes**: Where data is processed and Spark drivers/executors are launched

Compute _planes_ was used in the plural form because two subtypes can be distinguished based on the environment of their computational resources:
- **Serverless Compute Plane**: Resources reside in Databricks's cloud account in the same region as the associated workspace. Services like _Serverless SQL Warehouse_ entirely rely on this plane.
- **Classic** (or Traditional) **Compute Plane**: Data processing tasks run on VMs that are started in virtual networks under the user's own cloud account. For example, _Job Compute_ was one of the first offerings by Databricks and its Spark clusters
  consist of AWS EC2, Azure VM, or Google Compute Engine instances.

This compute layer difference has also major implications for the associated service costs which are described in the next [article](./costs.md).
Data teams do not directly interact with these compute planes, most of their work will involve the workspace UI which is a web application that, as depicted in the image, is hosted in the Databricks control plane.
Many account and workspace entities can also be contacted via REST [API](https://docs.databricks.com/api/workspace/introduction) or with the Databricks command-line interface.

## Databricks and Google Kubernetes Engine
In the overview [diagram](../resources/images/dbx-arch.png), Databricks on GCP occupies a dedicated area as it leverages Google's Kubernetes Engine which has architectural and cost implications: A new GCP Databricks workspace bootstraps a Kubernetes
cluster that will host the classic data plane, each workspace is associated with its own GKE cluster. A Spark cluster ist just a group of one or more nodes with a distinct K8s namespace in a workspace's GKE cluster.
Therefore, all Spark processes launched from the same workspace run in pods of the same K8s cluster. Each Spark driver and executor runs in a separate pod, a Virtual Machine instance hosts one Spark pod. <br>

Even if no workloads are executed, a system node pool is active at all times and incurs Compute Engine costs as multiple GCE instances run across multiple availability zones to support their workspace.
The official documentation [mentions](https://docs.gcp.databricks.com/en/getting-started/index.html#set-up-a-databricks-free-trial-and-first-workspace) per-workspace costs of "approximately $200/month, prorated to the days in the month that the GKE cluster runs", the next [article](./costs.md)
addresses such pricing aspects in more detail. As an automatic cost savings measure, any workspace GKE cluster gets deleted after five days of inactivity, i.e. when no Databricks clusters were launched during
that time period.

## Workspace Storage
Each Databricks workspace relies on at least one storage bucket/account in the object store of the user's cloud vendor. This resource hosts
- the DBFS root (explained below)
- the metastore root with the workspace's default catalog when Unity Catalog is enabled (see the Unity Catalog section)
- system data holding workspace and usage artefacts like logs, notebook revisions, or job results

**DBFS** (Databricks File System) is an HDFS-like abstraction over cloud-based object stores. The **DBFS root** is created during workspace deployment and mounted into a workspace and its clusters.
It is/was the underlying default storage container for some workspace operations and items like user files uploaded via the UI. The DBFS root also serves as the default location for the Hive metastore.
All users can access the root which was one of the reasons for the introduction of the Unity Catalog with its more fine-grained access policies. Several DBFS data access patterns
and directories (along with the Hive metastore) are now deprecated in favour of the functionality of the Unity Catalog.

## Unity Catalog

Coming soon

## Databricks Clusters
In the Big Data world, developers write distributed programs that are executed on a cluster of virtual or physical machines. Databricks supports multiple languages (Scala, SQL, Java, Python, R) in
several formats and offers different types of clusters referred to by the umbrella term **Compute**. On the Databricks platform, a **cluster** is a collection of one or more virtual machine instances. In
**Spark clusters**, one of these VM instances is special as the driver that executes the main program loop runs on it. The other instances are optional and host Spark executors that perform the actual
data processing work. One instance hosts one Spark executor.

The various cluster types can be configured and started in different ways such as via APIs, the Databricks CLI, or directly from the workspace UI. A developer's
code artefact (like a notebook or JAR file) can often be executed on more than one Compute type, they are orthogonal. For example:
- Notebooks can be attached to All-Purpose and Job Compute. Pythonic and SQL notebook cells can run on Serverless Compute and SQL cells on SQL Warehouses.
- Job Compute tasks can be created from different assets such as notebooks, Python scripts/wheel packages, JAR files, SQL queries, DLT pipelines, or dbt projects.
- Several Job Compute task types (such as Python scripts or JARs) can be executed on an All-Purpose cluster. SQL tasks can only be run on SQL warehouses.

## Compute Types

## Serverless versus Traditional Compute

## Jobs versus All-Purpose Compute

## SQL Warehouses