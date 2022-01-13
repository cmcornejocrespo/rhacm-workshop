# Exercise 2 - Managing an existing cluster using Advanced Cluster Management

In this exercise you manage the existing cluster on the Red Hat Advanced Cluster Management stack - `local-cluster` and you will create a new cluster and import an existing one. You will attach labels to the clusters, visualize their resources and perform updates to the OpenShift Platform.

**Note:** AWS as the infrastructure provider will be used for this exercise.
## 2.1 Create a new cluster


Before creating new cluster you need to add the credentials for the cloud you want to provision the cluster in.

Navigate to **Cluster Lifecycle**

1. Click on the left panel menu -> **Credentials** -> **Add credential**.
* Select Credential type (Cloud provider credentials)
* Credential name
* Namespace (where you want to store the credentials)
* Base DNS domain (DNS wildcard for your cluster)
* Access Key ID
* Secret access key
* Pull secret (from cloud.redhat.com)
* SSH public key
2. Finish the process by clicking **Add**

Create a new cluster

1. Navigate to **Cluster Lifecycle** -> **Create cluster**.
* Select infrastructure provider
* Add the cluster name
* Add the DNS Base domain
* Add the release image
* Add additional labels (optional)
* Select the Region
* Select the Control plane pool
* Select the Worker pool
* Select the network type and the network settings
* Select the proxy (if applies)
* Configure the automation (if applies)
## 2.2 Import an existing cluster (via API token)

1. Navigate to **Cluster Lifecycle** -> **Import cluster**.
* Add the cluster name
* Add additional labels (optional)
* Select server URL and API token as the import mode
* Add the Server URL
* Add the API Token

2. Make sure that all of the agent pods are up and running on the cluster.

```sh
<managed cluster> $ oc get pods -n open-cluster-management-agent
NAME                                         	READY   STATUS	RESTARTS   AGE
klusterlet-645d98d7d5-hnn2z                  	1/1 	Running   0      	46m
klusterlet-registration-agent-66fdc479cf-ltlx6   1/1 	Running   0      	46m
klusterlet-registration-agent-66fdc479cf-qnhzj   1/1 	Running   0      	46m
klusterlet-registration-agent-66fdc479cf-t8x5n   1/1 	Running   0      	46m
klusterlet-work-agent-6b8b99b899-27ht9       	1/1 	Running   0      	46m
klusterlet-work-agent-6b8b99b899-95dkr       	1/1 	Running   1      	46m
klusterlet-work-agent-6b8b99b899-vdp9r       	1/1 	Running   0      	46m

<managed cluster> $ oc get pods -n open-cluster-management-agent-addon
NAME                                                  READY   STATUS RESTARTS   AGE
klusterlet-addon-appmgr-64695b8d7b-q7jdg               2/2 	Running   0      	44m
klusterlet-addon-certpolicyctrl-75df48646-j5zpc        2/2 	Running   0      	44m
klusterlet-addon-iampolicyctrl-6867d45954-xv9c2        2/2 	Running   0      	44m
klusterlet-addon-operator-f85df9c7f-lb2cq              1/1 	Running   0      	45m
klusterlet-addon-policyctrl-config-policy-6fd8d88f5-chjls 1/1 	Running   0  44m
klusterlet-addon-policyctrl-framework-7544f4678-kwbz9  4/4 	Running   0      	44m
klusterlet-addon-search-7b697f899f-5vqwl               2/2 	Running   0      	44m
klusterlet-addon-workmgr-7f48869cd6-mqq8g               2/2 	Running   0      	44m
```
## 2.3 Hibernate an existing cluster
1. Navigate to **Cluster Lifecycle** -> Select the cluster -> **Actions** -> **Hibernate clusters**.
2. Click on **Hibernate**.
## 2.3.1 Hibernate an existing cluster
1. Navigate to **Cluster Lifecycle** -> Select the cluster -> **Actions** -> **Resume clusters**.
2. Click on **Resume**.
## 2.4 Upgrade the cluster using Advanced Cluster Management

**NOTE**: Do this exercise towards the end of the day. The upgrading process may take up to an hour to complete.

1. Change the **channel version** on the cluster from stable-**4.x** to stable-**4.x+1**.
2. Upgrade the cluster using Red Hat Advanced Cluster Management.

## 2.5 Analyzing managed clusters

In this exercise you will be using the Red Hat Advanced Cluster Management portal to analyze the managed cluster’s resources.

1. Navigate to **Cluster Lifecycle** -> Click on any cluster -> **Overview** -> Details.
2. Find out what is the cloud provider of the managed cluster.
2. Find out the number of nodes that make up the managed cluster. How many CPUs does each node have?
3. Using the Search, check out whether all users can provision new projects on local-cluster (check if the **self-provisioners** ClusterRoleBinding has the system:authenticated:oauth group associated with it).
4. Using the Search, check what **channel version** is associated with local-cluster (stable / candidate / fast) - (Search for **kind:ClusterVersion** CR).
5. Using the Search:
*   Check the port number that the **alertmanager-main-0** pod listens on local-cluster (can be found using the pod logs and pod resource definition).
*   Check the full path of the **alertmanager-main-0** pod configuration file (can be found using the pod logs and pod resource definition).
6. Create a custom search for storageClassName:gp2 and save it as "pvc's provisioned using gp2".