---
title: Deployment guidance
titleSuffix: SQL Server big data clusters
description: Learn how to deploy [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ver15.md)] (preview) on Kubernetes.
author: MikeRayMSFT 
ms.author: mikeray
ms.reviewer: mihaelab
ms.date: 08/28/2019
ms.topic: conceptual
ms.prod: sql
ms.technology: big-data-cluster
---

# How to deploy [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)] on Kubernetes

[!INCLUDE[tsql-appliesto-ssver15-xxxx-xxxx-xxx](../includes/tsql-appliesto-ssver15-xxxx-xxxx-xxx.md)]

SQL Server big data cluster is deployed as docker containers on a Kubernetes cluster. This is an overview of the setup and configuration steps:

- Set up a Kubernetes cluster on a single VM, cluster of VMs, or in Azure Kubernetes Service (AKS).
- Install the cluster configuration tool **azdata** on your client machine.
- Deploy a SQL Server big data cluster in a Kubernetes cluster.

[!INCLUDE [Limited public preview note](../includes/big-data-cluster-preview-note.md)]

## Install SQL Server 2019 big data tools

Before deploying a SQL Server 2019 big data cluster, first [install the big data tools](deploy-big-data-tools.md):

- **azdata**
- **kubectl**
- **Azure Data Studio**
- **SQL Server 2019 extension**

## <a id="prereqs"></a> Kubernetes prerequisites

[!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)] require a minimum Kubernetes version of at least v1.13 for both server and client (kubectl).

> [!NOTE]
> Note that the client and server Kubernetes versions should be within +1 or -1 minor version. For more information, see [Kubernetes release notes and version skew SKU policy)](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/release/versioning.md#supported-releases-and-component-skew).

### <a id="kubernetes"></a> Kubernetes cluster setup

If you already have a Kubernetes cluster that meets the above prerequisites, then you can skip directly to the [deployment step](#deploy). This section assumes a basic understanding of Kubernetes concepts.  For detailed information on Kubernetes, see the [Kubernetes documentation](https://kubernetes.io/docs/home).

You can choose to deploy Kubernetes in any of three ways:

| Deploy Kubernetes on: | Description | Link |
|---|---|---|
| **Azure Kubernetes Services (AKS)** | A managed Kubernetes container service in Azure. | [Instructions](deploy-on-aks.md) |
| **Multiple machines (kubeadm)** | A Kubernetes cluster deployed on physical or virtual machines using **kubeadm** | [Instructions](deploy-with-kubeadm.md) |
| **Minikube** | A single-node Kubernetes cluster in a VM. | [Instructions](deploy-on-minikube.md) |

> [!TIP]
> You can also script the deployment of AKS and a big data cluster in one step. For more information, see how to do this in a [python script](quickstart-big-data-cluster-deploy.md) or an Azure Data Studio [notebook](deploy-notebooks.md).

### Verify Kubernetes configuration

Run the **kubectl** command to view the cluster configuration. Ensure that kubectl is pointed to the correct cluster context.

```bash
kubectl config view
```

After you have configured your Kubernetes cluster, you can proceed with the deployment of a new SQL Server big data cluster. If you are upgrading from a previous release, please see [How to upgrade [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)]](deployment-upgrade.md).

## <a id="deploy"></a> Deployment overview

Most big data cluster settings are defined in a JSON deployment configuration file. You can use a default deployment profile for AKS, `kubeadm`, or `minikube` or you can customize your own deployment configuration file to use during setup. For security reasons, authentication settings are passed via environment variables.

The following sections provide more details on how to configure your big data cluster deployments as well as examples of common customizations. Also, you can always edit the custom deployment configuration file using an editor like VS Code for example.

## <a id="configfile"></a> Default configurations

Big data cluster deployment options are defined in JSON configuration files. You can start your customization of the cluster deployment from the built-in deployment profiles with default settings for dev/test environments:

| Deployment profile | Kubernetes environment |
|---|---|
| **aks-dev-test** | Azure Kubernetes Service (AKS) |
| **kubeadm-dev-test** | Multiple machines (kubeadm) |
| **minikube-dev-test** | minikube |

You can deploy a big data cluster by running **azdata bdc create**. This prompts you to choose one of the default configurations and then guides you through the deployment.

The first time you run `azdata` you must include `--accept-eula=yes` to accept the end user license agreement (EULA).

```bash
azdata bdc create --accept-eula=yes
```

In this scenario, you are prompted for any settings that are not part of the default configuration, such as passwords. 

> [!IMPORTANT]
> The default name of the big data cluster is **mssql-cluster**. This is important to know in order to run any of the **kubectl** commands that specify the Kubernetes namespace with the `-n` parameter.

## <a id="customconfig"></a> Custom configurations

It is also possible to customize your own deployment configuration profile. You can do this with the following steps:

1. Start with one of the standard deployment profiles that match your Kubernetes environment. You can use the  **azdata bdc config list** command to list them:

   ```bash
   azdata bdc config list
   ```

1. To customize your deployment, create a copy of the deployment profile with the **azdata bdc config init** command. For example, the following command creates a copy of the **aks-dev-test** deployment configuration files in a target directory named `custom`:

   ```bash
   azdata bdc config init --source aks-dev-test --target custom
   ```

   azdata
   > The `--target` specifies a directory that contains the configuration files, **bdc.json** and **control.json**, based on the `--source` parameter.

1. To customize settings in your deployment configuration profile, you can edit the deployment configuration file in a tool that is good for editing JSON files, such as VS Code. For scripted automation, you can also edit the custom deployment profile using **azdata bdc config** command. For example, the following command alters a custom deployment profile to change the name of the deployed cluster from the default (**mssql-cluster**) to **test-cluster**:  

   ```bash
   azdata bdc config replace --config-file custom/bdc.json --json-values "metadata.name=test-cluster"
   ```
   
   > [!TIP]
   > You can also pass in the cluster name at deployment time using the *--name* parameter for *azdata create bdc* command. The parameters in the command have precedence over the values in the configuration files.

   > A useful tool for finding JSON paths is the [JSONPath Online Evaluator](https://jsonpath.com/).

   In addition to passing key-value pairs, you can also provide inline JSON values or pass JSON patch files. For more information, see [Configure deployment settings for big data clusters](deployment-custom-configuration.md).

1. Then pass the custom configuration file to **azdata bdc create**. Note that you must set the required [environment variables](#env), otherwise you will be prompted for the values:

   ```bash
   azdata bdc create --config-profile custom --accept-eula yes
   ```

> For more information on the structure of a deployment configuration file, see the [Deployment configuration file reference](reference-deployment-config.md). For more configuration examples, see [Configure deployment settings for big data clusters](deployment-custom-configuration.md).

## <a id="env"></a> Environment variables

The following environment variables are used for security settings that are not stored in a deployment configuration file. Note that Docker settings except credentials can be set in the configuration file.

| Environment variable | Requirement |Description |
|---|---|---|
| **CONTROLLER_USERNAME** | Required |The username for the cluster administrator. |
| **CONTROLLER_PASSWORD** | Required |The password for the cluster administrator. |
| **MSSQL_SA_PASSWORD** | Required |The password of SA user for SQL master instance. |
| **KNOX_PASSWORD** | Required |The password for Knox **root** user. Note than in a basic authentication setup only user supported for Knox is **root**.|
| **ACCEPT_EULA**| Required for first use of `azdata`| Set to "yes". When set as an environment variable, it applies EULA to both SQL Server and `azdata`. If not set as environment variable, you can include `--accept-eula=yes` in the first use of `azdata` command.|
| **DOCKER_USERNAME** | Optional | The username to access the container images in case they are stored in a private repository. See the [Offline deployments](deploy-offline.md) topic for more details on how to use a private Docker repository for big data cluster deployment.|
| **DOCKER_PASSWORD** | Optional |The password to access the above private repository. |

These environment variables must be set prior to calling **azdata bdc create**. If any variable is not set, you are prompted for it.

The following example shows how to set the environment variables for Linux (bash) and Windows (PowerShell):

```bash
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>
export MSSQL_SA_PASSWORD=<password>
export KNOX_PASSWORD=<password>
export ACCEPT_EULA=yes
```

```PowerShell
SET CONTROLLER_USERNAME=admin
SET CONTROLLER_PASSWORD=<password>
SET MSSQL_SA_PASSWORD=<password>
SET KNOX_PASSWORD=<password>
```

> [!NOTE]
> You must use **root** user for Knox gateway with the above password. **root** is the only user supported for in this basic authentication (username/password) setup. For SQL Server master, username provisioned to be used with the above password is **sa**.


After setting the environment variables, you must run `azdata bdc create` to trigger the deployment. This example uses the cluster configuration profile created above:

```bash
azdata bdc create --config-profile custom --accept-eula yes
```

Please note the following guidelines:

- Make sure you wrap the password in double quotes if it contains any special characters. You can set the **MSSQL_SA_PASSWORD** to whatever you like, but make sure the password is sufficiently complex and don't use the `!`, `&` or `'` characters. Note that double quotes delimiters work only in bash commands.
- The **SA** login is a system administrator on the SQL Server master instance that gets created during setup. After creating your SQL Server container, the **MSSQL_SA_PASSWORD** environment variable you specified is discoverable by running `echo $MSSQL_SA_PASSWORD` in the container. For security purposes, change your SA password as per best practices documented [here](../linux/quickstart-install-connect-docker.md#sapassword).

## <a id="unattended"></a> Unattended install

For an unattended deployment, you must set all required environment variables, use a configuration file, and call `azdata bdc create` command with the `--accept-eula yes` parameter. The examples in the previous section demonstrate the syntax for an unattended installation.

## <a id="monitor"></a> Monitor the deployment

During cluster bootstrap, the client command window will output the deployment status. During the deployment process, you should see a series of messages where it is waiting for the controller pod:

```output
Waiting for cluster controller to start.
```

In 15 to 30 minutes, you should be notified that the controller pod is running:

```output
Cluster controller endpoint is available at 11.111.111.11:30080.
Cluster control plane is ready.
```

> [!IMPORTANT]
> The entire deployment can take a long time due to the time required to download the container images for the components of the big data cluster. However, it should not take several hours. If you are experiencing problems with your deployment, see [Monitoring and troubleshoot [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)]](cluster-troubleshooting-commands.md).

When the deployment finishes, the output notifies you of success:

```output
Cluster deployed successfully.
```

> [!TIP]
> The default name for the deployed big data cluster is `mssql-cluster` unless modified by a custom configuration.

## <a id="endpoints"></a> Retrieve endpoints

After the deployment script has completed successfully, you can obtain the IP addresses of the external endpoints for the big data cluster using the following steps.

1. After the deployment, find the IP address of the controller endpoint either from the deployment standard output or by looking at the EXTERNAL-IP output of the following **kubectl** command:

   ```bash
   kubectl get svc controller-svc-external -n <your-big-data-cluster-name>
   ```

   > [!TIP]
   > If you did not change the default name during deployment, use `-n mssql-cluster` in the previous command. **mssql-cluster** is the default name for the big data cluster.

1. Log in to the big data cluster with [azdata login](reference-azdata.md). Set the **--controller-endpoint** parameter to the external IP address of the controller endpoint.

   ```bash
   azdata login --controller-endpoint https://<ip-address-of-controller-svc-external>:30080 --controller-username <user-name>
   ```

   Specify the username and password that you configured for the controller (CONTROLLER_USERNAME and CONTROLLER_PASSWORD) during deployment.

1. Run [azdata bdc endpoint list](reference-azdata-bdc-endpoint.md) to get a list with a description of each endpoint and their corresponding IP address and port values. 

   ```bash
   azdata bdc endpoint list -o table
   ```

   The following list shows sample output from this command:

   ```output
   Description                                             Endpoint                                                   Ip              Name               Port    Protocol
   ------------------------------------------------------  ---------------------------------------------------------  --------------  -----------------  ------  ----------
   Gateway to access HDFS files, Spark                     https://11.111.111.111:30443                               11.111.111.111  gateway            30443   https
   Spark Jobs Management and Monitoring Dashboard          https://11.111.111.111:30443/gateway/default/sparkhistory  11.111.111.111  spark-history      30443   https
   Spark Diagnostics and Monitoring Dashboard              https://11.111.111.111:30443/gateway/default/yarn          11.111.111.111  yarn-ui            30443   https
   Application Proxy                                       https://11.111.111.111:30778                               11.111.111.111  app-proxy          30778   https
   Management Proxy                                        https://11.111.111.111:30777                               11.111.111.111  mgmtproxy          30777   https
   Log Search Dashboard                                    https://11.111.111.111:30777/kibana                        11.111.111.111  logsui             30777   https
   Metrics Dashboard                                       https://11.111.111.111:30777/grafana                       11.111.111.111  metricsui          30777   https
   Cluster Management Service                              https://11.111.111.111:30080                               11.111.111.111  controller         30080   https
   SQL Server Master Instance Front-End                    11.111.111.111,31433                                       11.111.111.111  sql-server-master  31433   tcp
   HDFS File System Proxy                                  https://11.111.111.111:30443/gateway/default/webhdfs/v1    11.111.111.111  webhdfs            30443   https
   Proxy for running Spark statements, jobs, applications  https://11.111.111.111:30443/gateway/default/livy/v1       11.111.111.111  livy               30443   https
   ```

You can also get all the service endpoints deployed for the cluster by running the following **kubectl** command:

```bash
kubectl get svc -n <your-big-data-cluster-name>
```

### Minikube

If you are using minikube, you need to run the following command to get the IP address you need to connect to. In addition to the IP, specify the port for the endpoint you need to connect to.

```bash
minikube ip
```

## <a id="status"></a> Verify the cluster status

After deployment, you can check the status of the cluster with the [azdata bdc status show](reference-azdata-bdc-status.md) command.

```bash
azdata bdc status show
```

> [!TIP]
> To run the status commands, you must first log in with the **azdata login** command, which was shown in the previous endpoints section.

The following shows sample output from this command:

```output
Bdc: ready                                                                                                                                                                                                          Health Status:  healthy
 ===========================================================================================================================================================================================================================================
 Services: ready                                                                                                                                                                                                     Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Servicename    State    Healthstatus    Details

 sql            ready    healthy         -
 hdfs           ready    healthy         -
 spark          ready    healthy         -
 control        ready    healthy         -
 gateway        ready    healthy         -
 app            ready    healthy         -


 Sql Services: ready                                                                                                                                                                                                 Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 master          ready    healthy         StatefulSet master is healthy
 compute-0       ready    healthy         StatefulSet compute-0 is healthy
 data-0          ready    healthy         StatefulSet data-0 is healthy
 storage-0       ready    healthy         StatefulSet storage-0 is healthy


 Hdfs Services: ready                                                                                                                                                                                                Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 nmnode-0        ready    healthy         StatefulSet nmnode-0 is healthy
 storage-0       ready    healthy         StatefulSet storage-0 is healthy
 sparkhead       ready    healthy         StatefulSet sparkhead is healthy


 Spark Services: ready                                                                                                                                                                                               Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 sparkhead       ready    healthy         StatefulSet sparkhead is healthy
 storage-0       ready    healthy         StatefulSet storage-0 is healthy


 Control Services: ready                                                                                                                                                                                             Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 controldb       ready    healthy         -
 control         ready    healthy         -
 metricsdc       ready    healthy         DaemonSet metricsdc is healthy
 metricsui       ready    healthy         ReplicaSet metricsui is healthy
 metricsdb       ready    healthy         StatefulSet metricsdb is healthy
 logsui          ready    healthy         ReplicaSet logsui is healthy
 logsdb          ready    healthy         StatefulSet logsdb is healthy
 mgmtproxy       ready    healthy         ReplicaSet mgmtproxy is healthy


 Gateway Services: ready                                                                                                                                                                                             Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 gateway         ready    healthy         StatefulSet gateway is healthy


 App Services: ready                                                                                                                                                                                                 Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 appproxy        ready    healthy         ReplicaSet appproxy is healthy
```

You can also get more detailed status with the following commands:

- [azdata bdc control status show](reference-azdata-bdc-control-status.md) will return health status for all components associated with the control management service
```
azdata bdc control status show
```
Sample output:
```output
Control: ready                                                                                                                                                                                                      Health Status:  healthy
 ===========================================================================================================================================================================================================================================
 Resources: ready                                                                                                                                                                                                    Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 controldb       ready    healthy         -
 control         ready    healthy         -
 metricsdc       ready    healthy         DaemonSet metricsdc is healthy
 metricsui       ready    healthy         ReplicaSet metricsui is healthy
 metricsdb       ready    healthy         StatefulSet metricsdb is healthy
 logsui          ready    healthy         ReplicaSet logsui is healthy
 logsdb          ready    healthy         StatefulSet logsdb is healthy
 mgmtproxy       ready    healthy         ReplicaSet mgmtproxy is healthy
```

- **azdata bdc sql status show** will return health status for all resources that have a SQL Server service
```
azdata bdc sql status show
```
Sample output:
```output
Sql: ready                                                                                                                                                                                                          Health Status:  healthy
 ===========================================================================================================================================================================================================================================
 Resources: ready                                                                                                                                                                                                    Health Status:  healthy
 -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Resourcename    State    Healthstatus    Details

 master          ready    healthy         StatefulSet master is healthy
 compute-0       ready    healthy         StatefulSet compute-0 is healthy
 data-0          ready    healthy         StatefulSet data-0 is healthy
 storage-0       ready    healthy         StatefulSet storage-0 is healthy
```

> [!IMPORTANT]
> When using **--all** parameter the output from these commands contain URLs to Kibana and Grafana dashboards for more detailed analysis.

In addition to using **azdata**, you can also use Azure Data Studio to find both endpoints and status information. For more information about viewing cluster status with **azdata** and Azure Data Studio, see [How to view the status of a big data cluster](view-cluster-status.md).

## <a id="connect"></a> Connect to the cluster

For more information on how to connect to the big data cluster, see [Connect to a SQL Server big data cluster with Azure Data Studio](connect-to-big-data-cluster.md).

## Next steps

To learn more about big data cluster deployment, see the following resources:

- [Configure deployment settings for big data clusters](deployment-custom-configuration.md)
- [Perform an offline deployment of a SQL Server big data cluster](deploy-offline.md)
- [Workshop: Microsoft [!INCLUDE[big-data-clusters-2019](../includes/ssbigdataclusters-ss-nover.md)] Architecture](https://github.com/Microsoft/sqlworkshops/tree/master/sqlserver2019bigdataclusters)
