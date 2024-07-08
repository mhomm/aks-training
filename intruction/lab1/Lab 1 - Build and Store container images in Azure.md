# Lab 1 - Build and Store container images in Azure

## Introduction

In this Lab you will deploy an **Azure Container Registry (ACR)** to your subscription and leverage **ACR Tasks** to build your container images directly from the Cloud. Your images will then be stored in your ACR for latter use.

* Expected Lab duration: 30 /40 minutes

## Challenge yourself (OPTIONAL)

If you want to challenge yourself, try to perform the following actions on your own:

1. Follow Exercise 1 to 3 to deploy the **Azure Container Registry** with correct settings and clone the application repository
2. Try to build the `yadaweb:1.0` and `yadaapi:1.0` images from the sources of the application using ACR Tasks (make sure you use these names and tags)

## Exercise 1: Deploy Azure Container Registry

To create an **Azure Container Registry** you can choose to do it using the portal or Azure CLI (other methods are available such as Terraform, Bicep, Pulumi, PowerShell)

### Using Azure Portal

1. On the Azure Portal, in the search bar on the top of the screen search for `container registries` (you may find it using only registry or containers too) then click on `Container Registries` in the `Services` category
2. Click on the `Create` button
3. Fill the `Basic` Tab with the following information (replace **X** with your TRAINEE number):
   | Setting            | Value                       |
   |--------------------|-----------------------------|
   | Subscription       |Training_B5NWJ001_AFKLM_AKS    |
   | Resource Group     | rg-akstraining**X**         |
   | Registry Name      | **PREFIX**acr**X**          |
   | Location           | West Europe                 |
   | Availability Zones | Checked                     |
   | SKU                | Premium                     |
4. Click `Next: Networking`
5. Keep default settings (in real life, you will likely use a Private access to ensure all connections to your ACR stay within your network)
6. Click `Next: Encryption`
7. Keep default settings, the ACR will still be encrypted but using Microsoft managed keys
8. Click `Next: Tags`
9. Tags are use for governance of your Azure Resources (such as charge back, identify resource owner etc. we will not use them in the labs)
10. Click `Next: Review + Create`
11. Click `Create`

### Using Azure CLI

1. Open **Azure Cloud Shell**
2. Execute the following commands to create the **Azure Container Registry**:

   ```azcli
   ACR_NAME="${PREFIX}acr${TRAINEE_NB}"

   az acr create -n ${ACR_NAME} \
              -g ${RESOURCE_GROUP} \
              --sku Premium \
              --zone-redundancy Enabled
   ```

## Exercise 2: Explore Azure Container Registry (OPTIONAL)

Once the ACR has been deployed, you can explore its configuration and content using Azure Portal.

1. On the Azure Portal, in the search bar on the top of the screen search for `container registries` as before (or directly type the name of your registry and skip next step)
2. Click on the container registry you have created earlier
3. On the center part you start with the overview of the service:
   1. The general information of your registry (general azure resource information, Login server to use when connecting to the registry, its SKU)
   2. Information regarding its usage (consumed space)
   3. Metrics information (number of pull/push per sec, task runs etc.)
   4. On the top bar you can also change the SKU of your registry by clicking on the `Update` button
4. On the left pane you have access to different panes to further configure the service or to have a look at its content:
   * Activity Log: What happened in the service (for audit purpose)
   * Access Control (IAM): That's where you'll configure who can access your container registry (such as developer groups or CI tools for pushing images or AKS cluster to pull images from the registry)
   * Access Keys: If you want to use the legacy connection method to your ACR using a username and a key (equivalent to a password)
   * Identity: If your ACR needs to connect to other Azure Services when performing tasks
   * Networking: If you want to reconfigure the networking options or want to restrict access
   * Microsoft Defender for Cloud: if you want to enable integration with this service to scan your stored images for vulnerabilities
   * Locks: If you want to ensure nobody can delete/change configuration of your ACR
   * **Repositories**: The list of your content in your registry (images, helm charts etc.)
   * WebHooks: Every time something happens in your registry you can trigger an action in an external service
   * **Tasks**: The list of defined tasks and all runs that occurred in your registry (will use it in next exercises)
   * Metrics: Graph/Dashboard of what happened in your registry
   * Diagnostic Settings: if you want to export the logs and metrics of the service to somewhere else

## Exercise 3: Clone the Demo Application in your environment

The different labs will use a demo application made by Microsoft: **YADA** (Yet Another Demo App). If you want to learn mode about this application, you can go to their [GitHub Repository](https://github.com/microsoft/YADA).

The application consists of a web frontend displaying a web page and a backend providing APIs. We can also connect the backend to a Database and perform queries.

In this exercise we will clone the repository in our Cloud Shell environment so you have files locally available.

1. Open **Azure Cloud Shell**
2. Execute the following commands (remember later to go to the YADA folder if you're not in it when you reopen Cloud Shell):

   ```sh
   git clone https://github.com/microsoft/YADA.git
   cd YADA
   ```

3. You can explore the code using linux commands such as `ls`, `cat`, or open the `code` editor in the YADA folder and explore the different files in it. You can also directly browse the repository with your browser.
4. Have a look on the `DockerFile` files in the `web` and `api` folders

## Exercise 4: Build Container images using ACR Tasks

Now we have the YADA Application source code and the Azure Container Registry deployed, we can build the 2 images: **web** and **api**

In order to build the different images, execute the following commands:

```azcli
ACR_NAME="${PREFIX}acr${TRAINEE_NB}"

az acr build -r ${ACR_NAME} -g ${RESOURCE_GROUP} -t yadaweb:1.0 web
az acr build -r ${ACR_NAME} -g ${RESOURCE_GROUP} -t yadaapi:1.0 api
```

These commands will do the following:

* Take the source code of your different images (in the web and api folders) and compress them in a `.tar.gz` file
* Upload the compress file to the ACR Runners in the cloud
* Launch a Run Task in ACR to build the images
* Tag each images with their name (`yadaweb` and `yadaapi`) and the `1.0` tag
* Upload the images in your ACR

After the commands succeed, you can have a look on the repositories in your ACR on the Azure Portal:

1. On the Azure Portal, in the search bar on the top of the screen search for `container registries` as before (or directly type the name of your registry and skip next step)
2. Click on the container registry you have created earlier
3. On the left pane, click on `Repositories`
4. You can find the two images created earlier
5. If you click on each of them you can have a look on the different tags and metadata information
6. On the left pane, click on `Tasks` then in the main part the `Runs` Tab
7. For each image you can have a look on their corresponding build status and logs by clicking on their respective line

You can also access the status and logs using these commands in Cloud Shell:

```azcli
az acr task list-runs --registry ${ACR_NAME} -o table
az acr task logs --registry ${ACR_NAME} --run-id cb1
az acr task logs --registry ${ACR_NAME} --run-id cb2

az acr task logs --registry ${ACR_NAME} --image yadaapi:1.0
az acr task logs --registry ${ACR_NAME} --image yadaweb:1.0
```
