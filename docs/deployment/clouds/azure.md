# Microsoft Azure

## Required types of resources

- [Azure FIles](https://azure.microsoft.com/en-us/products/storage/files)
- [AKS](https://azure.microsoft.com/en-us/products/kubernetes-service/)
    - [Azure Disk](https://azure.microsoft.com/en-us/products/storage/disks/)
    - [Azure App Gateway](https://azure.microsoft.com/en-us/products/application-gateway/)
    - [Azure DNS](https://azure.microsoft.com/en-us/products/dns/)

!!! tip "Limitation"

    At the moment this setup can only work with a Azure Files mount in a pods.

## Deployment

You can use any convenient way for you to create resources in Azure.

- ARM templates
- Terraform
- Pulumi
- Manual creation

## Recommended steps for AKS configuration

1. Deploy [AzureFile CSI Driver](https://github.com/kubernetes-sigs/azurefile-csi-driver).

    1. Create [Secret, PV and PVC](https://learn.microsoft.com/en-us/azure/aks/azure-csi-files-storage-provision).

## Configuration examples

<details><summary>Secret example.</summary>
```yaml
apiVersion: v1
data:
  azurestorageaccountkey: xxxx
  azurestorageaccountname: xxxx
kind: Secret
metadata:
  name: azure-secret
  namespace: odm
type: Opaque
```
</details>

<details><summary>PersistentVolume example.</summary>
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-azurefile
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azurefile-csi
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=0
    - gid=0
    - mfsymlinks
    - cache=strict
    - nosharesock
  csi:
    driver: file.csi.azure.com
    readOnly: false
    volumeHandle: 1111-xxxx-23
    volumeAttributes:
      resourceGroup: xxxxx
      shareName: xxxxx
    nodeStageSecretRef:
      name: azure-secret
      namespace: odm
```
</details>

<details><summary>PersistentVolumeClaim example.</summary>
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-azurefile
  namespace: odm
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  volumeName: pv-azurefile
  storageClassName: azurefile-csi
```
</details>
