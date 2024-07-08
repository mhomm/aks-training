# Lab 0 - Connect to Azure Environment

## Connect to Azure Portal

1. Open an Incognito window in your favorite browser:
   * Edge: `New InPrivate Window (Ctrl+ Shift + N)`
   * Chrome: `New Incognito Window (Ctrl + Shift + N)`
   * Firefox: `New Private Window (Ctrl + Shift + P)`
2. Go to [Azure Portal](https://portal.azure.com)
3. Log in to the Portal using your training credentials:
   * Username: `afklm-traineex@avanadeacademyfrance.onmicrosoft.com` (replacing X with your assigned traineennumber)
   * Password: Password provided by your trainer
4. Do not setup MFA: Click on `Skip for now (14 days until this is required)`
5. Stay signed in by clicking on `Yes`
6. Click on `Maybe later` if the portal ask you for an Azure tour
7. [OPTIONAL] If Azure Portal is not in English and you want to switch to this language :
   * On the top bar, click on the gear wheel :gear: icon
   * On the left menu, click on `Language + region`
   * Select your desired language
   * Click on `Save` at the bottom

## Set up Azure Cloud Shell

1. On the top menu bar, click on the `Shell` icon
2. Click on `Bash` shell (the one we will use during the training)
3. Click on `Show advanced settings` as the storage account is already created
4. Use the following settings:
   | Setting            | Value                                                  |
   |--------------------|--------------------------------------------------------|
   | Subscription       | Training_B5NWJ001_AFKLM_AKS                             |
   | Cloud Shell region | West Europe                                            |
   | Resource Group     | Use Existing: rg-cloudshell-weu         |
   | Storage Account    | Use Existing: csastrainees                             |
   | File share         | Enter: trainee`X` (replace X with your Student Number) |
5. Click on `Attach Storage`

## Discover Azure Cloud Shell

Using Azure Cloud Shell you can launch several Linux commands such as `ls`, `cd`, `more`, `cat`, `man`, `vim` but you also have access to additional programs such as `terraform`, `kubectl`, `helm`, `ansible`, `git`, `az` (to interact with Azure resources)

You can also access a code editor (like VS Code) using the `code` command:

* Use right-click to access code drop down actions (to `Save` or `Close Editor`)
* You can also use the shortcuts:
  * `Ctrl+S` to Save
  * `Ctrl+Q` to Close Editor

If you want to access Cloud Shell in a Full Screen Experience, you can go to [Cloud Shell](https://shell.azure.com), otherwise you can maximise or minimize the frame.

## Set up common variables used during training

In order to simplify further commands, we will setup variables that will be used in the next labs:

1. Open `.bashrc` file using the code editor:

   ```sh
   code .bashrc
   ```

2. Add the following lines at the end of the file:

   ```sh
   export PREFIX=afk 
   export TRAINEE_NB=X # your assigned student number
   export RESOURCE_GROUP="trainee-khrd6m-${TRAINEE_NB}_rg"
   export LOCATION=westeurope
   ```

3. Save using `Ctrl+S` then quit using `Ctrl+Q`
4. Reload the `.bashrc` file using the command:

   ```sh
   source .bashrc
   ```


   
