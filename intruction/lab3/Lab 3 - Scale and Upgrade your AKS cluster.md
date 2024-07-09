# Lab 3 - Scale and Upgrade your AKS cluster

## Introduction

In this Lab you will implement **Horizontal Pod Autoscaler** (HPA) for the backend API, launch π calculation that will trigger new pod creation. Then you will upgrade your  **Azure Kubernetes Service (AKS) cluster** to a newer Kubernetes version.

* Expected Lab duration: 30 minutes

## Challenge yourself (OPTIONAL)

If you want to challenge yourself, try to perform the following actions on your own:

1. Implement an `HorizontalPodAutoscaler` object on API deployment to have between `1` and `3` pods when CPU usage is above `50%`
2. Test your `HPA` by stressing the API by calculating `50000` Pi decimals on YADA web page (at the bottom)
3. Follow instructions to upgrade your cluster and see what is happening on your infrastructure

## Exercise 1: Implement Horizontal Pod Autoscaler for the API Deployment

To implement HPA on the API Deployment you first need to make sure the `metrics-server` pods are running. By default, AKS has it enabled.

1. Open **Azure Cloud Shell**
2. Execute the following command to check `metrics-server` pods are running:

   ```sh
   kubectl get deployments metrics-server -n kube-system
   ```

3. Execute the following command to see how much resources your API pod is currently using:

   ```sh
   kubectl top pod -l run=api
   ```

4. Execute the following command to download the **Horizontal Pod Autoscaler** configuration file

   ```sh
   wget https://raw.githubusercontent.com/mhomm/aks-training/main/yaml/lab3/api-hpa.yml
   ```

5. Open the files in the code editor to understand the configuration (target used, how many pods can run at minimum and maximum, what will trigger this autoscaler)
6. Deploy the yaml file into your cluster using the following commands:

   ```sh
   kubectl apply -f api-hpa.yml
   ```

7. Have a look at the HPA status and conditions met using the following command:

   ```sh
   kubectl describe hpa api-hpa
   ```

8. Now we will watch HPA behavior when we have load on our API. Use the following command to see changes in your HPA status (note the `--watch` parameter to listen to changes of our resource)

   ```sh
   kubectl get hpa api-hpa --watch
   ```

9. Navigate to the YADA web page you deployed in previous lab
10. At the bottom of the page, you have the `Calculate number Pi` section, enter `50000` in the `Number of digits to calculate` and click `Submit` button
11. You should see after some time changes in your Cloud Shell output (from step 8.):
    * Increase at `TARGETS` column (XX%/50%)
    * Additional Pod being created
    * Decrease at `TARGETS` column (we have one pod using 100% and another one using 0 so the average is 50)
12. Exit the watch using `Ctrl+C`.
13. Use the following command to see you have now 2 pods and how much CPU they are using:

    ```sh
    kubectl get pod -l run=api
    kubectl top pod -l run=api
    ```

## Exercise 2: Upgrade your AKS Cluster to a newer version

In this exercise we will upgrade our AKS cluster to a newer minor version. For example, if your cluster is currently running `1.23`, we will migrate to `1.24`. Remember you can only migrate one minor version at a time, we cannot directly switch to `1.25`.
We will first migrate the **Control Plane** of our cluster, and then upgrade the **Default Node Pool** and see what is happening in our cluster.

### Using Azure Portal

1. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` as before (or directly type the name of your AKS cluster and skip next step)
2. Click on the AKS cluster you have created earlier
3. On the left pane click on `Cluster Configuration`
4. You will see in the center pane the `Kubernetes version` and click on `Upgrade version`
5. Select the next minor version, check `Upgrade control plane only` and then click on `Save`
6. Wait for the **Control Plane** upgrade to finish (will take a few minutes).

Now the **Control Plane** is upgraded you can start to upgrade the nodes of your **Default Node Pool**

1. On the left pane click on `Node Pools`
2. You can see the node pools of your cluster and their corresponding version
3. Click on the line of `agentpool` pool and click on top on `Upgrade Kubernetes`
4. It will open a new pane on the right, choose the latest Kubernetes version and click `Apply`
5. It will perform the upgrade, in the next part you will see how things go inside your cluster and on Azure Portal

### Using Azure CLI

1. Open **Azure Cloud Shell**
2. You can assess the version of your Kubernetes Cluster (Control Plane) using these commands:

   ```sh
   AKS_CLUSTER="${PREFIX}aks${STUDENT_NB}"

   # Get all details for server and client
   kubectl version -o yaml

   # Get only the server version
   kubectl version -o json | jq '.serverVersion.gitVersion'

   # Using the az command
   az aks show \
    -g ${RESOURCE_GROUP} \
    -n ${AKS_CLUSTER} \
    --query 'currentKubernetesVersion'
   ```

3. Execute the following commands to get the available Kubernetes versions:

   ```azcli
   #Note the versions available
   az aks get-upgrades -g ${RESOURCE_GROUP} -n ${AKS_CLUSTER} -o table
   ```

4. Perform the Control Plane upgrade (replace `KUBERNETES_VERSION`) with the version you identified in last step

   ```azcli
   KUBERNETES_VERSION=1.XX.YY

   #Perform control plane upgrade
   az aks upgrade \
    -g ${RESOURCE_GROUP} \
    -n ${AKS_CLUSTER} \
    --kubernetes-version ${KUBERNETES_VERSION} \
    --control-plane-only

   ```

5. Confirm the different prompts (using the `y` key)

Now the **Control Plane** is upgraded you can start to upgrade the nodes of your **Default Node Pool**

1. You can assess the version of your nodes using theses commands:

   ```sh
   # Get nodes version using kubectl command
   kubectl get nodes

   # Using the az command
   az aks show \
    -g ${RESOURCE_GROUP} \
    -n ${AKS_CLUSTER} \
    --query "agentPoolProfiles[].{name: name, version:currentOrchestratorVersion}" \
    -o table
   ```

2. Upgrade your node pool using these command lines (the `--no-wait` parameter is used to launch the upgrade in the background allowing us to launch the next commands):

   ```azcli
   # Set the KUBERNETES_VERSION if not already defined
   # KUBERNETES_VERSION=1.XX.YY

   POOL_NAME="agentpool"

   # Using the az command
   az aks nodepool upgrade \
    -g ${RESOURCE_GROUP} \
    --cluster-name ${AKS_CLUSTER} \
    --name ${POOL_NAME} \
    --kubernetes-version ${KUBERNETES_VERSION} \
    --no-wait
   ```

3. The node pool will now be upgraded

### Check Node upgrade process

During the Node Pool upgrade, the different nodes will be upgraded to the target Kubernetes version. The underlying Virtual Machines will be rebuilt. In this part you will see how things are going one.

1. Open **Azure Cloud Shell**
2. Use the following command to follow all the changes in your nodes (note the `--watch` parameter to live follow what is happening. You will see nodes being added/removed. Otherwise you can execute the command without this parameter multiple times until the cluster been fully upgraded):

   ```sh
   # Get nodes version using kubectl command
   kubectl get nodes -o wide --watch
   ```

3. On the portal, if you are not already on your cluster resource on Azure Portal do the next 2 steps otherwise, skip them
4. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` as before (or directly type the name of your AKS cluster and skip next step)
5. Click on the AKS cluster you have created earlier
6. On the left pane, click on `Properties`
7. Click on the link in the `Infrastructure resource group` section. It should look like `MC_${RESOURCE_GROUP]_{AKS_CLUSTER}_westeurope`
8. You will be in the resource group where Microsoft deployed all the underlying resources being used by your cluster and manged by Microsoft. You should not perform any actions here but reading information or having a look at it never killed someone.
9. Click on the resource named like `aks-agentpool-XXXXXXX-vmss` (with `XXXXXXX` being a random value). This is the `Virtual Machine Scale Set`, a group of identically configured Virtual Machines running your Node Pool
10. In the `Status` value in the center, you will it is currently upgrading. Depending on how fast you performed these actions you can also see the `Size` value showing `(2 -> 3 instances)`
11. Click on the left pane on `Instances`, you can here see the nodes being upgraded, added, removed. Click on `Refresh` from time to time.

Voilà. Your cluster is running a newer version of Kubernetes. In real life you must of course validate the health of your cluster before and after the upgrade, ensure all your applications are compatible etc.
