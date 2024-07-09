# Lab 4 - Namespaces and Quotas

## Introduction

In this Lab you configure the YADA application in different namespaces. The goal is to mimic an application developed by 2 independent teams: one in charge of the `web` frontend and one in charge of the `api` backend.

You will then implement resource quotas on the `web` namespace to ensure you don't scale above the configured limits.

* Expected Lab duration: 30 minutes

## Challenge yourself (OPTIONAL)

If you want to challenge yourself, try to perform the following actions on your own:

1. Delete the resources created in earlier labs
2. Create 2 namespaces: `web` and `api` (be careful, you'll need additional permissions on your cluster, have a look at the `Role assignments` in the `Access Control (IAM)` of your cluster)
3. Redeploy the YADA application microservices in their respective namespace
4. Validate you can access the application frontend and that it can reach the backend
5. Create a `ResourceQuota` resource applied on the `web` namespace that restrict resources created in your namespace to the following:

   | Quota           | Value |
   |-----------------|-------|
   | CPU Requests    | 1     |
   | CPU Limits      | 2     |
   | Memory Requests | 1Gi   |
   | Memory Limits   | 2Gi   |

6. Scale the `web Deployment` above the quota and look at the errors in corresponding resources.
7. Scale back to `1` pod

## Exercise 1: Clean environment

The current YADA application has been deployed in the `default` namespace. This is not a good practice but this is normal, you never heard of namespaces before.

You need to clean your cluster first.

1. Open **Azure Cloud Shell**
2. Execute the following commands to remove all resources in the environment (note: you shouldn't do that in production. This command only delete "common" kind of resources, but this is enough for our simple application):

   ```sh
   kubectl delete all --all
   ```

## Exercise 2: Add required permissions to create namespaces

In your current AKS environment, your user account has all privileges except `creating namespaces and manage quotas` which is unfortunate. To solve this, you need to add some additional permissions.

You can perform this exercise either using the Azure Portal or you can use Azure CLI (choose one of them)

### Azure Portal

1. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` as before (or directly type the name of your AKS cluster and skip next step)
2. Click on the AKS cluster you have created earlier
3. On the left pane click on `Access Control (IAM)`
4. In the middle, click on `Add role assignment`
5. Select `Azure Kubernetes Service RBAC Cluster Admin` role (make sure you select this one and not another as the names are quite close)
6. Click on `Next`
7. Click on `+ Select members`
8. Search for your user account on the right pane
9. Click on your user account (it should now appear in the bottom part of the pane)
10. Click on `Select`
11. Click on `Review + assign`
12. On the confirm page, click on `Review + assign` again

### Azure CLI

1. Open **Azure Cloud Shell**
2. Execute the following commands to add your account as cluster administrator:

   ```azcli
   AKS_CLUSTER="${PREFIX}aks${STUDENT_NB}"

   # First you need to get your user id
   USER_ID=$(az ad signed-in-user show --query id -o tsv)
   
   # You can have a look on the content of this command to know more about your user account:
   az ad signed-in-user show

   # Then you need your AKS Cluster Resource ID
   AKS_RESOURCE_ID=$(az aks show -g ${RESOURCE_GROUP} -n ${AKS_CLUSTER} --query 'id' -o tsv)
   # Have a look at the variable content:
   echo $AKS_RESOURCE_ID

   # Assign permission:
   az role assignment create --assignee $USER_ID --role "Azure Kubernetes Service RBAC Cluster Admin" --scope $AKS_RESOURCE_ID
   ```

## Exercise 3: Redeploy our application in namespaces

In this exercise you will create the two namespaces and then redeploy the different resources in their corresponding namespace (`Deployment`,`Service`,`ConfigMap`)

1. Open **Azure Cloud Shell**
2. Create two namespaces (you can do that using `kubectl` command in imperative way or create YAML files as usual)

   ```sh
   kubectl create ns web
   kubectl create ns api
   ```

3. Redeploy the different YAML files in their corresponding namespaces:

   ```sh
   for f in web*.yml; do kubectl apply -f $f -n web; done
   for f in api*.yml; do kubectl apply -f $f -n api; done
   ```

4. Go back on your YADA Web App, you can in the central section `Information retrieved from API http://api:8080` see that all fields are empty. This is because the front end cannot communicate with the api. Now your application is deployed in different namespaces, you must update the used URL to reach your APIs.
5. To solve this issue, you need to edit your web deployment:

   ```sh
   code web-deployment.yml
   ```

6. Change L25 of the file to `value: "http://api.api:8080"`
7. Save the file (using `Ctrl+S`) and exit the code editor (using `Ctrl+Q`)
8. Re-apply your yml file:

   ```sh
   kubectl apply -f web-deployment.yml -n web
   ```

9. Ensure your deployment succeeded:

   ```sh
   kubectl rollout history -n web deployment web
   kubectl rollout status -n web deployment web
   kubectl get deployments.apps -n web web
   ```

10. Go back on your YADA Web App, you can see in the central section `Information retrieved from API http://api.api:8080` that all fields are now filled with your API answers.

You have successfully redeployed your application in namespaces.

Don't forget to use the `-n` parameter to your `kubectl` commands to ensure you are in the correct namespace. You can also use `-A` or `--all-namespaces` switches to target all the namespaces (useful for `get` or `describe` sub-commands).

## Exercise 3: Implement Resource quotas

Now you have redeploy the application in different namespaces you'll ensure they cannot use more resources the cluster administrator allocated to them.

To enforce this you will deploy a `ResourceQuota` resource on your `web` namespace and see what happens when you try to go beyond your quota.

1. First, have a look at your current `web` namespace:

   ```sh
   kubectl describe ns web
   ```

2. Download the `ResourceQuota` resource from the repository and apply it:

   ```sh
   wget https://raw.githubusercontent.com/mhomm/aks-training/main/yaml/lab4/web-resourcequota.yml
   kubectl apply -n web -f web-resourcequota.yml
   ```

3. Your namespace now enforce quotas. You can see how much your current pods use using the same command as the first step.
4. Try to scale your web front end to multiple pods and see what happens:

   ```sh
   kubectl scale -n web deployment web --replicas 2
   kubectl describe ns web
   kubectl get all -n web
   kubectl scale -n web deployment web --replicas 3
   kubectl describe ns web
   kubectl get all -n web
   ```

5. The `CPU Requests` are above the quota when you scale to `3` pods. You can notice only `2` pods deployed.
6. Have a look at the `ReplicaSet` object of the web deployment to see the corresponding errors (in the `Events` section):

   ```sh
   kubectl describe replicaset -n web
   ```

7. Scale back to `1` pod:

   ```sh
   kubectl scale -n web deployment web --replicas 1
   ```
