# Lab 8 : Persistent Data


## Exercice 1: Create a persistent volume claim

A persistent volume claim (PVC) automatically provisions storage based on a storage class. In this case, a PVC can use one of the precreated storage classes to create a standard or premium Azure managed disk.

1. Create a file named `azure-pvc.yaml` and copy in the following manifest. The claim requests a disk named `azure-managed-disk` that's *5 GB* in size with *ReadWriteOnce* access. The *managed-csi* storage class is specified as the storage class.

      ```yaml
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
          name: azure-managed-disk
      spec:
        accessModes:
        - ReadWriteOnce
        storageClassName: managed-csi
        resources:
          requests:
            storage: 5Gi
      ```



2. Create the persistent volume claim using the [`kubectl apply`][kubectl-apply] command and specify your *azure-pvc.yaml* file.

      ```bash
      kubectl apply -f azure-pvc.yaml
      ```

      The output of the command resembles the following example:

      ```output
      persistentvolumeclaim/azure-managed-disk created
      ```

### Use the persistent volume

After you create the persistent volume claim, you must verify it has a status of `Pending`. The `Pending` status indicates it's ready to be used by a pod.

1. Verify the status of the PVC using the `kubectl describe pvc` command.

    ```bash
    kubectl describe pvc azure-managed-disk
    ```

    The output of the command resembles the following condensed example:

    ```output
    Name:            azure-managed-disk
    Namespace:       default
    StorageClass:    managed-csi
    Status:          Pending
    [...]
    ```

2. Create a file named `azure-pvc-disk.yaml` and copy in the following manifest. This manifest creates a basic NGINX pod that uses the persistent volume claim named *azure-managed-disk* to mount the Azure Disk at the path `/mnt/azure`. For Windows Server containers, specify a *mountPath* using the Windows path convention, such as *'D:'*.

    ```yaml
    kind: Pod
    apiVersion: v1
    metadata:
      name: mypod
    spec:
      containers:
        - name: mypod
          image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
          volumeMounts:
            - mountPath: "/mnt/azure"
              name: volume
              readOnly: false
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: azure-managed-disk
    ```

3. Create the pod using the [`kubectl apply`][kubectl-apply] command.

      ```bash
       kubectl apply -f azure-pvc-disk.yaml
      ```
4. You now have a running pod with your Azure Disk mounted in the `/mnt/azure` directory. Check the pod configuration using the [`kubectl describe`][kubectl-describe] command.

      ```bash
       kubectl describe pod mypod
      ```

## Exercice 2: Statically provision a volume

This section provides guidance for cluster administrators who want to create one or more persistent volumes that include details of Azure Disks for use by a workload.

### Static provisioning parameters for a persistent volume

The following table includes parameters you can use to define a persistent volume.

|Name | Meaning | Available Value | Mandatory | Default value|
|--- | --- | --- | --- | ---|
|volumeHandle| Azure disk URI | `/subscriptions/{sub-id}/resourcegroups/{group-name}/providers/microsoft.compute/disks/{disk-id}` | Yes | N/A|
|volumeAttributes.fsType | File system type | `ext4`, `ext3`, `ext2`, `xfs`, `btrfs` for Linux, `ntfs` for Windows | No | `ext4` for Linux, `ntfs` for Windows |
|volumeAttributes.partition | Partition number of the existing disk (only supported on Linux) | `1`, `2`, `3` | No | Empty (no partition) </br>- Make sure partition format is like `-part1` |
|volumeAttributes.cachingMode | [Disk host cache setting][disk-host-cache-setting] | `None`, `ReadOnly`, `ReadWrite` | No  | `ReadOnly`|

### Create an Azure disk

When you create an Azure disk for use with AKS, you can create the disk resource in the **node** resource group. This approach allows the AKS cluster to access and manage the disk resource. If you instead create the disk in a separate resource group, you must grant the Azure Kubernetes Service (AKS) managed identity for your cluster the `Contributor` role to the disk's resource group.

1. Identify the resource group name using the [`az aks show`][az-aks-show] command and add the `--query nodeResourceGroup` parameter.

    ```azurecli-interactive
    az aks show --resource-group myResourceGroup --name myAKSCluster --query nodeResourceGroup -o tsv
    ```

    The output of the command resembles the following example:
   
    ```output
    MC_myResourceGroup_myAKSCluster_eastus
    ```

1. Create a disk using the [`az disk create`][az-disk-create] command. Specify the node resource group name and a name for the disk resource, such as *myAKSDisk*. The following example creates a *20*GiB disk, and outputs the ID of the disk after it's created. If you need to create a disk for use with Windows Server containers, add the `--os-type windows` parameter to correctly format the disk.

    ```azurecli-interactive
    az disk create \
      --resource-group MC_myResourceGroup_myAKSCluster_eastus \
      --name myAKSDisk \
      --size-gb 20 \
      --query id --output tsv
    ```

    The disk resource ID is displayed once the command has successfully completed, as shown in the following example output. You use the disk ID to mount the disk in the next section.

    ```output
    /subscriptions/<subscriptionID>/resourceGroups/MC_myAKSCluster_myAKSCluster_eastus/providers/Microsoft.Compute/disks/myAKSDisk
    ```

### Mount disk as a volume

1. Create a *pv-azuredisk.yaml* file with a *PersistentVolume*. Update `volumeHandle` with disk resource ID from the previous step. For Windows Server containers, specify *ntfs* for the parameter *fsType*.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      annotations:
        pv.kubernetes.io/provisioned-by: disk.csi.azure.com
      name: pv-azuredisk
    spec:
      capacity:
        storage: 20Gi
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: managed-csi
      csi:
        driver: disk.csi.azure.com
        volumeHandle: /subscriptions/<subscriptionID>/resourceGroups/MC_myAKSCluster_myAKSCluster_eastus/providers/Microsoft.Compute/disks/myAKSDisk
        volumeAttributes:
          fsType: ext4
    ```

2. Create a *pvc-azuredisk.yaml* file with a *PersistentVolumeClaim* that uses the *PersistentVolume*.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: pvc-azuredisk
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
      volumeName: pv-azuredisk
      storageClassName: managed-csi
    ```

3. Create the *PersistentVolume* and *PersistentVolumeClaim* using the [`kubectl apply`][kubectl-apply] command and reference the two YAML files you created.

    ```bash
    kubectl apply -f pv-azuredisk.yaml
    kubectl apply -f pvc-azuredisk.yaml
    ```

4. Verify your *PersistentVolumeClaim* is created and bound to the *PersistentVolume* using the `kubectl get pvc` command.

    ```bash
    kubectl get pvc pvc-azuredisk
    ```

    The output of the command resembles the following example:

    ```output
    NAME            STATUS   VOLUME         CAPACITY    ACCESS MODES   STORAGECLASS   AGE
    pvc-azuredisk   Bound    pv-azuredisk   20Gi        RWO                           5s
    ```

5. Create an *azure-disk-pod.yaml* file to reference your *PersistentVolumeClaim*. For Windows Server containers, specify a *mountPath* using the Windows path convention, such as *'D:'*.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
        name: mypod
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        volumeMounts:
          - name: azure
            mountPath: /mnt/azure
      volumes:
        - name: azure
          persistentVolumeClaim:
            claimName: pvc-azuredisk
    ```

6. Apply the configuration and mount the volume using the [`kubectl apply`][kubectl-apply] command.

    ```bash
    kubectl apply -f azure-disk-pod.yaml
    ```