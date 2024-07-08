# Lab 2 - Kubernetes Basics

## Introduction

In this Lab you will deploy your **Azure Kubernetes Service (AKS) cluster** to your subscription and deploy the **YADA** application in it.

* Expected Lab duration: 45 minutes

## Challenge yourself (OPTIONAL)

If you want to challenge yourself, try to perform the following actions on your own:

1. After following Exercises 1 to 4, try to create YAML files to deploy both `web` and `api` images (from your registry) in your cluster. You can find information on `Deployment` resource on official Kubernetes website. `api` and `web` images can be configured using environment variables. Go to `README.md` files in the respective folders to learn more about these variables
   * Make sure you configure the correct environment variable to establish communication between `web` front end and `api` backend. Don't worry about `SQL*` or `KV*` environment variables for this lab.
   * Your pods must be in `Running` state
2. Scale your deployments to `2` pods then go back to `1`
3. Try to get and describe the deployed resources to understand more (have a look at step by step to learn more)
4. Expose the `web` deployment to the outside world using a `LoadBalancer` service
5. Expose the `api` deployment internally using a `ClusterIp` service
6. Configure the `BACKGROUND` environment variable using a `ConfigMap` object you attach on `web` pods. The value is an hexadecimal number looking like `#ffccee` (choose whatever you want)

## Exercise 1: Deploy Kubernetes Service (AKS) cluster

To create an **Azure Kubernetes Service (AKS) cluster** you can choose to do it using the portal or Azure CLI (other methods are available such as Terraform, Bicep, Pulumi, PowerShell)

### Using Azure Portal

1. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` (you may find it using other terms such as aks too) then click on `Kubernetes Services` in the `Services` category
2. Click on the `Create` button and select `Create a Kubernetes Cluster`
3. Fill the `Basic` Tab with the following information (replace **X** with your TRAINEE number):
   | Setting                      | Value                       |
   |------------------------------|-----------------------------|
   | Subscription                 | Training_Gallia-Training    |
   | Resource Group               | trainee-khrd6m-**X**_rg        |
   | Cluster preset configuration | Dev/Test ($)                |
   | Kubernetes cluster name      | **PREFIX**aks**X**          |
   | Region                       | West Europe                 |
   | Availability Zones           | 1,2,3                       |
   | Kubernetes version           | Stick with the default one  |
   | API server availability      | 99.5% Optimize for cost     |
   | Automatic upgrade            | Disabled                    |
4. Configure the Primary node pool with these settings:
   | Setting                      | Value                       |
   |------------------------------|-----------------------------|
   | Node size                    | Standard B4ms               |
   | Scale method                 | Auto scale                  |
   | Node count range             | 2-3                         |
5. Click `Next: Node pools`
6. Keep default settings (in real life, you will have multiple node pools)
7. Click `Next: Access`
8. Authentication and Authorization: select `Azure AD authentication with Azure RBAC`
9. Click `Next: Networking`
10. Update **only** the following settings (details will be explained in the corresponding module):
   | Setting                      | Value                       |
   |------------------------------|-----------------------------|
   | Network configuration        | Azure CNI                   |
   | Network policy               | Azure                       |
11. Click `Next: Integrations` and configure
    * Container Registry: select the container registry you created in previous lab
    * Azure Policy: `Enabled`
12. Click `Next: Advanced`
    * Enable secret store CSI  driver: `Checked`
13. Click `Next: Tags`
14. Click `Next: Review + Create`
15. Click `Create`

### Using Azure CLI

1. Open **Azure Cloud Shell**
2. Execute the following commands to create the different resources:

   ```azcli
   ACR_NAME="${PREFIX}acr${TRAINEE_NB}"

   VNET="vnet-akstraining${TRAINEE_NB}"
   VNET_ADDRESS_PREFIX=10.224.0.0/12
   VNET_SUBNET_AKS=aks-subnet
   VNET_SUBNET_AKS_PREFIX=10.224.0.0/16

   az network vnet create -g ${RESOURCE_GROUP} \
                       -n ${VNET} \
                       --address-prefix ${VNET_ADDRESS_PREFIX} \
                       --subnet-name ${VNET_SUBNET_AKS} \
                       --subnet-prefix ${VNET_SUBNET_AKS_PREFIX} 

   VNET_SUBNET_AKS_ID=$(az network vnet subnet show \
                           --resource-group ${RESOURCE_GROUP} \
                           --vnet-name ${VNET} \
                           --name ${VNET_SUBNET_AKS} \
                           --query id \
                           -o tsv)
   
   AKS_CLUSTER="${PREFIX}aks${TRAINEE_NB}"
   AZURE_DEFAULT_AD_TENANTID=$(az account show --query tenantId --output tsv)

   # az aks get-versions -l ${LOCATION} -o table
   # AKS_VERSION=$(az aks get-versions -l ${LOCATION} -o tsv --query orchestrators[3].orchestratorVersion)

   AKS_VERSION=$(az aks get-versions --location westeurope --query "values[?isDefault!=null].version" -o tsv)

   # Create AKS cluster 
   az aks create --resource-group ${RESOURCE_GROUP} \
                 --name ${AKS_CLUSTER} \
                 --kubernetes-version ${AKS_VERSION} \
                 --enable-managed-identity \
                 --node-count 2 \
                 --enable-cluster-autoscaler \
                 --min-count 2 \
                 --max-count 5 \
                 --attach-acr ${ACR_NAME} \
                 --network-plugin azure \
                 --network-policy azure \
                 --service-cidr 10.0.0.0/16 \
                 --dns-service-ip 10.0.0.10 \
                 --vnet-subnet-id ${VNET_SUBNET_AKS_ID} \
                 --aad-tenant-id ${AZURE_DEFAULT_AD_TENANTID} \
                 --enable-aad \
                 --enable-azure-rbac \
                 --disable-local-accounts \
                 --enable-addons "azure-policy, azure-keyvault-secrets-provider" \
                 --generate-ssh-keys \
                 --node-osdisk-type Ephemeral \
                 --node-osdisk-size 30 \
                 --node-vm-size Standard_B4ms \
                 --vm-set-type VirtualMachineScaleSets \
                 --zones {1,2,3}
   ```

## Exercise 2: Explore Azure Kubernetes Services (OPTIONAL)

Once the AKS has been deployed, you can explore its configuration using Azure Portal.

1. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` as before (or directly type the name of your AKS cluster and skip next step)
2. Click on the AKS cluster you have created earlier
3. On the center part you start with the overview of the service:
   1. The general information of your cluster (general azure resource information and a summary of your cluster configuration)
   2. Information regarding node usage (in monitoring tab)
   3. on the top you can `Start`/`Stop` your cluster and `Connect` to it
4. On the left pane you have access to different panes to further configure the service or to have a look at its content:
   * Activity Log: What happened in the service (for audit purpose)
   * Access Control (IAM): That's where you'll configure who can access your cluster (such as administrator to manage your cluster but also as we will see later for your developer to connect to your cluster and manage applications)
   * Microsoft Defender for Cloud: if you want to enable integration with this service to scan your cluster for misconfiguration or vulnerable containers
   * Kubernetes resources (and underlying part): You can directly see (if you have network connectivity and permission) the content of your cluster and start basic troubleshooting from here without using command lines.
   * Settings (and underlying part): All the configuration we performed during cluster deployment (we will deep dive into that in the corresponding modules)
   * Metrics: Graph/Dashboard of what happened in your registry
   * Diagnostic Settings: if you want to export the logs and metrics of the service to somewhere else

## Exercise 3: Connect to your cluster

In this exercise we will connect to our cluster inside the Cloud Shell environment so we interact with it and then deploy our application in the next exercise.

1. If you are not already on your cluster resource on Azure Portal do the next 2 steps otherwise, skip them
2. On the Azure Portal, in the search bar on the top of the screen search for `kubernetes` as before (or directly type the name of your AKS cluster and skip next step)
3. Click on the AKS cluster you have created earlier
4. Click the `Connect` button and follow the instructions (the next steps are exactly the same as the one described in the portal but written with generic values)
5. Open **Azure Cloud Shell**
6. Execute the following commands :

   ```azcli
   AKS_CLUSTER="${PREFIX}aks${TRAINEE_NB}"

   az aks get-credentials -g ${RESOURCE_GROUP} -n ${AKS_CLUSTER}
   
   # List all deployments in all namespaces
   kubectl get deployments --all-namespaces=true
   ```

7. You will then be prompted to connect to your AKS cluster using your Azure AD credentials. This is expected. To do that you will have to open the Azure AD login page: [https://microsoft.com/devicelogin](https://microsoft.com/devicelogin)
8. On the cloud shell output you will see in the last line a code (looking like XXXXXXX), copy this code and paste it in the page you previously opened and click `Next`
9. Use the same login information as the one you used to connect to the portal. Don't forget to `skip` the MFA registration and `continue` until you have to `close` the window
10. If you come back to the Cloud Shell window, you will see all the deployments in your cluster.

## Exercise 4: Pimp your shell with autocompletion and kubecolor (RECOMMENDED)

During this training you will use a lot of `kubectl` commands. It can be difficult to remember all commands, parameters and some resources have random part in their names thus making them difficult to type. To ease that we can leverage autocomplete mechanisms and the shell will automatically enter the correct parameter, pod name or whatever or give a list we can refine later when you use the `Tab` (or double `Tab`) key on your keyboard.

Besides that, the `kubectl` output is all white by default, making it difficult to identify errors, important information and so on. A community tool has been created, called `kubecolor` that will enhance kubectl results to add colors.

While this is not mandatory, you can enable both of them during the training by using the following commands:

```sh
go install -ldflags="-X main.Version=latest" github.com/kubecolor/kubecolor/cmd/kubecolor@latest
cat <<EOF >> ~/.bashrc
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
command -v ~/go/bin/kubecolor >/dev/null 2>&1 && alias kubectl="~/go/bin/kubecolor"
command -v ~/go/bin/kubecolor >/dev/null 2>&1 && alias k=~/go/bin/kubecolor
complete -o default -F __start_kubectl k
EOF
source ~/.bashrc
```

## Exercise 5: Deploy the YADA Application in your Azure Kubernetes Services (AKS) Cluster

Now we are connected to our cluster and we are efficient at typing commands we can start deploying our application in it.

Reminder: Our application is composed of a web frontend and an API backend.

We will deploy containers for both of them, see how to customize our application, how to access it and do basic troubleshooting on it.

### Deploy the containers

1. Open **Azure Cloud Shell**
2. Execute the following commands to download the template deployment files :

   ```sh
   wget https://raw.githubusercontent.com
   wget https://raw.githubusercontent.com
   ```

3. You can start editing the files using the `code api-deployment.yaml` and `code web-deployment.yaml` commands
4. For each files you need to update the `<ACRNAME>` token with the name of your ACR (eg. `avaacr1`). This way, AKS will download the images from your Azure Container Registry you deployed in Lab 1.
5. Have a quick look at the other part of the files (such as the environment variables, the resource request/limits configured, labels etc.)
6. Deploy the yaml files into your cluster using the following commands:

   ```sh
   kubectl apply -f api-deployment.yml
   kubectl apply -f web-deployment.yml
   # You can also use one single command to apply both files:
   kubectl apply -f api-deployment.yml -f web-deployment.yml
   ```

7. You can have a look on what has been deployed using the following commands:

   ```sh
   kubectl get deployments
   kubectl get pods
   ```

   The pods must be in `Running` state.

8. You can get more info on the deployment and the pods

   ```sh
   kubectl describe deployments api
   kubectl describe pods nameofthepod # replace nameofthepod with the name of the pod from previous command, use autocomplete!
   ```

9. You can get the logs of pods

   ```sh
   kubectl logs nameofthepod # replace nameofthepod with the name of the pod from previous command, use autocomplete!
   kubectl logs -l run=api   # or use that to leverage selectors
   ```

10. You can execute commands from inside the pods for troubleshooting purpose:

    ```sh
    kubectl exec nameofthepod -it -- /bin/bash # replace nameofthepod with the name  of the pod from previous command, use autocomplete!
    # You are now inside the pod, not in Cloud Shell !
    top # It will display information about current processes, note there are only a few processes in a container ! Use Ctrl+C to exit top
    exit # to exit the container (or use Ctrl+D)

    kubectl exec nameofthepod -- env # to directly execute a command and exit (in this case the env command to display environment variables)
    ```

11. You can scale your deployment to have more instances

    ```sh
    kubectl scale deployment web --replica 2 
    kubectl get deployment web
    kubectl get pods -l run=web -o wide
    ```

### Configure Services and access your application

Now our application is deployed we need to configure services for 2 purposes:

* Allow user access to the web frontend using a `type: LoadBalancer` Service
* Allow internal communication between the web frontend and the api backend using a `type: ClusterIP` Service

1. Open **Azure Cloud Shell**
2. Execute the following commands to download the service files :

   ```sh
   wget https://raw.githubusercontent.com/jbpaux/aks-infra-training/main/yaml/lab2/api-service.yml
   wget https://raw.githubusercontent.com/jbpaux/aks-infra-training/main/yaml/lab2/web-service-lb.yml
   ```

3. Open the files in the code editor to understand the configuration (selector, type and ports)
4. Update on the web-service-lb.yml file the annotation to replace the **X** in `yadaTRAINEEX` by your TRAINEE number
5. Deploy these files into your cluster using the following commands:

   ```sh
   kubectl apply -f api-service.yml
   kubectl apply -f web-service-lb.yml
   ```

6. You can have a look on what has been deployed using these commands. Note the `EXTERNAL-IP` column of the web service. It should display a Public IP from Azure. If it is still `Pending`, wait for a few seconds.

   ```sh
   kubectl get services -o wide #or svc as a shortcut to list the services
   kubectl get endpoints #to see which IP are behind your services
   ```

7. You can now access your application using <http://EXTERNAL-IP> with the `EXTERNAL-IP` of your web frontend. You can also access your application using the DNS Prefix you configured in the yaml file: <http://yadastutentX.westeurope.cloudapp.azure.com> (replace **X** by your TRAINEE number)

### Use ConfigMaps and Secrets to configure your application

The background color of your web frontend can be dynamically configured. This is a good practice to externalize your application configuration outside of your container image.

In this part, we will create a `ConfigMap` resource with the expected environment variable used by the application and reconfigure the `Deployment` to use it. We could also just have added and environment variable to the existing list.

1. Open **Azure Cloud Shell**
2. Execute the following commands to download the `ConfigMap` template :

   ```sh
   wget https://raw.githubusercontent.com/jbpaux/aks-infra-training/main/yaml/lab2/web-configmap.yml
   ```

3. Open the file in the code editor to understand the configuration
4. Update if you want the value of the `BACKGROUND` variable. You can use a [Color Picker website](https://htmlcolorcodes.com/color-picker/) to select one.
5. Open with the code editor the `web-deployment.yml` file to reference the `ConfigMap`
6. Add the following lines in your file just before the `resources` section (L26):

   ```yaml
   envFrom:
   - configMapRef:
       name: web-configmap
   ```

   Make sure your indentation is correct (the envFrom must be at the same level as the `env` or `resources` section). Using `envFrom` we are importing all environment variables from the ConfigMap.

7. In case you have time, you can try to either move the `API_URL` environment variable into the ``ConfigMap` or reference each variable from the `ConfigMap` in your deployment.
8. Deploy these files into your cluster using the following commands:

   ```sh
   kubectl apply -f web-configmap.yml
   kubectl apply -f web-deployment.yml
   ```

9. Notice the `deployment/web` resource just changed
10. You can use the following command to see the deployment history:

    ```sh
    kubectl rollout status web
    kubectl rollout history web
    ```

11. You will notice the number of pods reverted to 1. This is expected as it is the value configured in the applied file.
