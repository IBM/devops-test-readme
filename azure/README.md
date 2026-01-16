# IBM DevOps Test Hub

IBM DevOps Test Hub brings together test data, test environments, and test runs and reports into a single, web-based browser for testers and non-testers. It is Kubernetes native and hence requires a cluster to run. If you do not have a cluster available we enable you to get started by providing scripts to provision a basic environment using K3s.

## Prerequisites






Setup an Azure Kubernetes cluster. Here is provided a reference implementation.

To install the product you will need cluster administrator privileges.

### Local Machine

Install [azure cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) then login, and connect to your subscription

Install [kubectl](https://learn.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-install-cli) and place on your PATH.

Install [helm v3.18.5 or later](https://helm.sh/docs/intro/install/) and place on your PATH.

Scripts have been validated using:
- [Git Bash](https://git-scm.com/downloads) on Windows
- Azure CLI version 2.71.0 (upgrade using `az upgrade --yes`)
- Azure Bicep CLI version 0.38.33 (upgrade using `az bicep upgrade`)

```bash
az login
az account set --subscription MyDept
```
### Subscription

To perform the complete installation of the product you need these roles:

* On the _Subscription_ both *Contributor* and *User Access Administrator* or simply *Owner*

Enable storage so that AKS can provision the `azurefile` RWX storage class
```bash
az provider register -n Microsoft.Storage
```
### Network design

This sample will provision on a single virtual network; Kubernetes with a VPN gateway to enable secure access and a Azure Container Registry used to cache images. For production deployments the VPN gateway should not be deployed and the virtual network should instead be peered with your organizations hub, linking with internal DNS zones.

The **172.16.0.0/20** private network block is used to minimize the chance of IP conflicts with internal networks (typically **10.0.0.0/8**) and home networks (typically **192.168.0.0/16**), however this is unlikely to be appropriate within your organization. Consult your Azure administrators regarding the network design.

### VPN Certificate

To enable secure communication with the product a certificate is required.

* Create a root and client [certificate](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-certificates-point-to-site).
* Export the root certificate naming it P2SRootCert.cer placing it in the same folder as main.bicep

### Deploy Template

The template found in the [helm chart](#install) provides minimal infrastructure to evaluate the product, however more compute should be considered on a production system for higher test throughput.

A dedicated nodepool is setup for test execution. To use this nodepool helm install uses the `values-dedicated-nodes.yaml` parameter.

When sizing the number of nodes you should confirm they are within your subscription [quota](https://portal.azure.com/#blade/Microsoft_Azure_Capacity/QuotaMenuBlade/myQuotas)

```bash
RESOURCE_GROUP=devops
INSTANCE=devops0
export IMAGE_REGISTRY=myorg$INSTANCE$(date -u +%Y%m%d).azurecr.io/devops

az group create -n $RESOURCE_GROUP --location eastus2
az deployment group create -g $RESOURCE_GROUP --template-file azure/main.bicep --parameters acrName=${IMAGE_REGISTRY%%.*} workVmMin=1 instanceName=$INSTANCE
az network vnet-gateway reset -g $RESOURCE_GROUP -n $INSTANCE
az network vnet-gateway vpn-client generate -g $RESOURCE_GROUP -n $INSTANCE
```

It takes about 23 minutes for the deployment to complete.

### Connect to VPN

Download the vpn client at the URL given by the _generate_ command, unzip and run `WindowsAmd64\VpnClientSetupAmd64.exe` confirming warnings.

Connect to the VPN via the Start Menu, typing _VPN Settings_ then _Connect_ on _devops0_ then following the sequence of prompts.

### kubeconfig

Configure kubectl to connect to the newly created cluster
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $INSTANCE --public-fqdn --overwrite-existing
kubectl get nodes
```

## Install

Fetch chart for install:
```bash
helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm --force-update
helm pull --untar ibm-helm/ibm-devops-prod --version 11.0.700
cd ibm-devops-prod
```
### Air gap / Local image registry

Move product images to the local registry:
```bash
ENTITLEMENT_REGISTRY_KEY= # from https://myibm.ibm.com/products-services/containerlibrary

PULL_ARGUMENTS="-g $RESOURCE_GROUP -u cp -p $ENTITLEMENT_REGISTRY_KEY" \
  bash lib/airgap/move-images.sh $IMAGE_REGISTRY cp.icr.io/cp
```


### Ingress controller



Install Emissary
```bash
export INGRESS_IP=172.16.0.14

export PLATFORM=azure

bash lib/ingress/main.sh

kubectl rollout status -n emissary deployment/emissary-ingress -w

# wait until the external-ip is assigned after a few minutes
kubectl get svc -n emissary -w emissary-ingress
```


If you see the `EXTERNAL-IP` is `<pending>` for a few minutes and this doesn't change to become $INGRESS_IP, you have a permissions problem which can be investigated using:
```bash
kubectl describe svc -n emissary emissary-ingress

  Warning  SyncLoadBalancerFailed  27s (x7 over 5m42s)  service-controller  Error syncing load balancer: failed to ensure load balancer: Retriable: false, RetryAfter: 0s, HTTPStatusCode: 403, RawError: Retriable: false, RetryAfter: 0s, HTTPStatusCode: 403, RawError: {"error":{"code":"AuthorizationFailed","message":"The client '00000000-0000-0000-0000-000000000000' with object id '00000000-0000-0000-0000-000000000000' does not have authorization to perform action 'Microsoft.Network/virtualNetworks/subnets/read' over scope '/subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/devops/providers/Microsoft.Network/virtualNetworks/vnet/subnets/subnet' or the scope is invalid. If access was recently granted, please refresh your credentials."}}
```
This means the identity used by AKS does not have the Network Contributor role necessary to access the subnet.

### Chart
```bash
NAMESPACE=devops-system
HELM_NAME=main

INGRESS_DOMAIN=$INGRESS_IP.nip.io
PASSWORD_SEED= # secure seed required to generate passwords - unrecoverable so keep it safe

RATIONAL_LICENSE_FILE=@rlks.localdomain

helm upgrade --install $HELM_NAME . -n $NAMESPACE \
  --create-namespace \
  --set global.domain=$INGRESS_DOMAIN \
  -f values-k8s.yaml \
  -f values-dedicated-nodes.yaml \
  --set global.persistence.rwxStorageClass=azurefile \
  --set imageRegistry=$IMAGE_REGISTRY \
  --set-literal passwordSeed=$PASSWORD_SEED \
  --set signup=true \
  --set rationalLicenseKeyServer=$RATIONAL_LICENSE_FILE \
  --set license=true
```
* When the ingress domain is accessible to untrusted parties, `signup` must be set to `false`.
* The password seed is used to generate default passwords and should be stored securely. Its required again to restore from a backup.


### Configuration

| Parameter                                      | Description | Default |
|------------------------------------------------|-------------|---------|
| `execution.ingress.hostPattern`                | Pattern used to generate hostnames so that running assets may be accessed via ingress. | PLATFORM specifc |
| `execution.nodePorts.enabled`                  | When `network.policy` is disabled, allow NodePorts to be used to access to running assets like virtual services. | true |
| `execution.priorityClassName`                  | The products dynamic workload pods will have this priorityClass. | '' |
| `execution.priorityClassValue`                 | When set a priorityClass named `execution.priorityClassName` is created with the set priority value. | |
| `global.domain`                                | The web address to expose the product on. For example `192.168.0.100.nip.io` | REQUIRED |
| `global.ibmCertSecretName`                     | Optionally used to terminate TLS and when `ingress.cert.selfSigned`, is used to verify trust of loopback connections. | ingress |
| `global.ibmImagePullSecret`                    | The docker-registry secret to pull images from the `imageRegistry`. | '' |
| `global.ibmImagePullUsername`                  | Username to pull images from the `imageRegistry`. | 'cp' |
| `global.ibmImagePullPassword`                  | Password to pull images from the `imageRegistry`. | '' |
| `global.privateCaBundleSecretName`             | Name of secret containing `ca.crt`, which lists certificates to trust (in PEM format). | '' |
| `rationalLicenseKeyServer`                     | Where floating licenses are hosted to entitle use of the product. For example `@ip-address` | '' |
| `imageRegistry`                                | The location of container images to use. See [move-images](lib/airgap/move-images.sh) | cp.icr.io/cp |
| `ingress.cert.create`                          | Create an self-signed certificate matching the ingress domain if none exists in secret `global.ibmCertSecretName`. | true |
| `ingress.cert.selfSigned`                      | If the ingress domain certificate is not signed by a globally trusted CA. | PLATFORM specifc |
| `keycloak.truststoreFileHostnameVerificationPolicy` | HTTPS hostname cerificate verifcation policy. ANY (hostname is not verified), WILDCARD (allows wildcards in subdomain names) or STRICT (the Common Name (CN) must match the hostname exactly). | WILDCARD |
| `license`                                      | Confirmation that the EULA has been accepted. For example `true` | false |
| `networkPolicy.egress.cidrs`                   | Network ranges to allow access to. This does not include access to github.com where helm test resources are stored. | [ 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 ] |
| `networkPolicy.egress.enable`                  | When `network.policy` is enabled create a rule to narrow egress from the product. | false |
| `networkPolicy.enabled`                        | Deny other software, installed in the cluster, access to the product. | true |
| `passwordSeed`                                 | The seed used to generate all passwords. | REQUIRED |
| `postgresql.migrate.enabled`                   | Enable Postgresql version migration on start when coming from v10.5.3. Migration is disabled to avoid an unnecessary image pull. | false |
| `priorityClassName`                            | The products pods (excluding dynamic workload) will have this priorityClass. | '' |
| `priorityClassValue`                           | When set a priorityClass named `priorityClassName` is created with the set priority value. | |
| `router.allowedOrigin`                         | A comma separated list of allowed origins for CORS. For example `*.domain.com,*.test.com,10.10.*.*`  | '' |
| `results.jaegerAgent`                          | The name of the service/host that execution engines write traces to. | '' |
| `results.jaegerDashboard`                      | The URL for where traces may be opened in a browser. | '' |
| `signup`                                       | Allow users to create their own accounts. (Setting also in realm under Login > User registration) | false |

## Upgrade

Upgrading from releases prior to v11.0.3 is not support - for older versions first upgrade to an intermediate release.

Before performing your upgrade RabbitMQ flags must be enabled on a running install:

```bash
kubectl exec -n $NAMESPACE $HELM_NAME-rabbitmq-0 -- rabbitmqctl enable_feature_flag all

```

If you are restoring from a quiesced snapshot, meaning no instance is running, you can instead delete the RabbitMQ data before installing:

```bash
kubectl delete pvc -n $NAMESPACE data-$HELM_NAME-rabbitmq-0

```

Before performing your upgrade backup your user data.


Install the product as [above](#chart).



## Backup

### Velero

Install [velero v14.0.1 or later](https://velero.io/docs/v1.14/basic-install/) and place on your PATH.




##### [Setup](https://github.com/vmware-tanzu/velero-plugin-for-microsoft-azure)

Deploy the velero backup template:

```bash
RESOURCE_GROUP=devops
INSTANCE=devops0
IDENTITY_NAME=velero
BLOB_CONTAINER=velero
AZURE_BACKUP_RESOURCE_GROUP=Velero_Backups
AZURE_STORAGE_ACCOUNT_ID="velero$(date -u +%s)"
MANAGED_CLUSTER_RESOURCE_GROUP=$(az aks show -g $RESOURCE_GROUP -n $INSTANCE -o tsv --query nodeResourceGroup)

az group create -n $AZURE_BACKUP_RESOURCE_GROUP --location eastus2
az deployment group create -g $MANAGED_CLUSTER_RESOURCE_GROUP --template-file azure/backup.bicep --parameters backupResourceGroupName=$AZURE_BACKUP_RESOURCE_GROUP identityName=$IDENTITY_NAME storageAccountId=$AZURE_STORAGE_ACCOUNT_ID blobContainerName=$BLOB_CONTAINER deployStorage=true
```

Create service account and cluster role binding:

```bash
AZURE_SUBSCRIPTION_ID=$(az account list --query '[?isDefault].id' -o tsv)
IDENTITY_CLIENT_ID="$(az identity show -g $MANAGED_CLUSTER_RESOURCE_GROUP -n $IDENTITY_NAME --subscription $AZURE_SUBSCRIPTION_ID --query clientId -o tsv)"

kubectl create namespace velero

cat <<EOF | kubectl apply -n velero -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
  name: velero
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: velero-cluster-admin
subjects:
- kind: ServiceAccount
  name: velero
  namespace: velero
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```

Establish federated identity credential between the identity and the service account issuer & subject:

```bash
SERVICE_ACCOUNT_ISSUER=$(az aks show --resource-group $RESOURCE_GROUP --name $INSTANCE --query oidcIssuerProfile.issuerUrl -o tsv)

az identity federated-credential create \
  --name kubernetes-federated-credential \
  --identity-name "$IDENTITY_NAME" \
  --resource-group "$MANAGED_CLUSTER_RESOURCE_GROUP" \
  --issuer "$SERVICE_ACCOUNT_ISSUER" \
  --subject system:serviceaccount:velero:velero
```

Create the velero credentials file that contains all the relevant environment variables:

```bash
cat << EOF  > ./credentials-velero
AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID
AZURE_RESOURCE_GROUP=$MANAGED_CLUSTER_RESOURCE_GROUP
AZURE_CLOUD_NAME=AzurePublicCloud
EOF
```

Install velero using the Azure plugin:

```bash
velero install \
    --provider azure \
    --plugins=velero/velero-plugin-for-microsoft-azure:v1.10.1 \
    --service-account-name velero \
    --pod-labels azure.workload.identity/use=true \
    --bucket $BLOB_CONTAINER \
    --secret-file ./credentials-velero \
    --backup-location-config useAAD="true",storageAccountURI="https://$AZURE_STORAGE_ACCOUNT_ID.blob.core.windows.net",resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,storageAccount=$AZURE_STORAGE_ACCOUNT_ID,subscriptionId=$AZURE_SUBSCRIPTION_ID \
    --snapshot-location-config apiTimeout=5m,resourceGroup=$AZURE_BACKUP_RESOURCE_GROUP,subscriptionId=$AZURE_SUBSCRIPTION_ID \
    --wait
```



Once velero install is complete, confirmed pods are running by using:

```bash
kubectl get pod -n velero
```

Backups with velero can now be created including targeted namespaces and _only_ persistent volumes:

```bash
BACKUP_NAME=$NAMESPACE
velero backup create $BACKUP_NAME --include-namespaces $NAMESPACE --snapshot-move-data --include-resources pvc,pv
```

Backup progress is monitored using:

```bash
velero backup describe $BACKUP_NAME --details
```

To restore a backup into a empty cluster - where a disaster scenario has occurred:

```bash
velero restore create --from-backup $BACKUP_NAME
```

Restore progress is monitored using:

```bash
velero restore describe $BACKUP_NAME --details
```

When restore of pvc's is complete, continue to install the product.


#### Data Migration

RabbitMQ will not run when migrating data to a different namespace. To enable re-initialization during install either:

- Skip the backup of RabbitMQ data: [`velero.io/exclude-from-backup=true`](https://velero.io/docs/v1.14/resource-filtering/#veleroioexclude-from-backuptrue)

```bash
kubectl label pvc -n $NAMESPACE data-$HELM_NAME-rabbitmq-0 velero.io/exclude-from-backup=true
```

- Delete the data post restore:

```bash
kubectl delete pvc -n $NAMESPACE data-$HELM_NAME-rabbitmq-0
```

## Verification

You can verify that the environment has completed startup with:
```bash
watch kubectl get pods -A
```
All the pods should change to a status of either Running or Complete.
```bash
bash lib/test/helm-diag.sh $HELM_NAME -n $NAMESPACE
```
## Uninstall


Delete the dynamic workload in the namespace:
```bash
kubectl delete statefulset,deployment,replicaset,job,pod --all -n $NAMESPACE
kubectl delete service,cm,secret -lapp.kubernetes.io/managed-by=$HELM_NAME.$NAMESPACE -n $NAMESPACE
```
Delete the product:
```bash
helm uninstall $HELM_NAME -n $NAMESPACE
```
The claims and persistent volumes that contain user data are not automatically be deleted. If you re-install the product these resources will be re-used if present.

To delete _EVERYTHING_, including user data contained in claims and persistent volumes
```bash
kubectl delete namespace $NAMESPACE
```
Note: This will hang if the namespace contains workload which has not terminated.


## Security Considerations

### Ingress

#### Firewall

The product loops back some requests via the ingress controller. It this is blocked by a firewall some pods will fail to transition to Running without it.

#### Trust of generated self signed certificate

When necessary the product generates a CA and certificate to terminate TLS. To fetch the generated CA so that it can be injected into other softwares trust stores, see the notes from:
```bash
helm status $HELM_NAME -n $NAMESPACE
```
#### Providing an new certificate

The helm install will automatically create a self-signed certificate to match the ingress domain. If you have a certificate signed by a CA you can use it instead by creating a secret of the following form before performing the install:
```bash
kubectl create namespace $NAMESPACE
kubectl create secret generic ingress -n $NAMESPACE \
  --type=kubernetes.io/tls \
  --from-file=ca.crt="./ca.crt" \
  --from-file=tls.crt="./tls.crt" \
  --from-file=tls.key="./tls.key"
```
Where tls.key is your private key, tls.crt is the certificate returned by your CA and ca.crt is the certificate of your CA (which signed your certificate). All these files are expected to be in PEM format. Note: the structure of the secret is consistent with those created by [cert-manager](https://cert-manager.io/docs/)

If the product is already installed. The secret can be replaced, but all the pods must be deleted or restarted manually to take effect.

#### Internal certificate expiry

After 365 days an internal certificate expires, causing the product to stop working. To resolve this:
```bash
kubectl delete secret emissary-ingress-webhook-ca -n emissary-system
kubectl delete pods -lapp.kubernetes.io/name=emissary-apiext -n emissary-system
```


### Trust of external self signed endpoints

The product only trusts certificates signed by recognized CAs. To trust additional CAs (for example your internal corporate CA), you must create a secret containing the additional CAs you wish to trust.

The certificate(s) must be in PEM format. You may list CAs in the ca.crt entry.
```bash
kubectl create secret generic -n $NAMESPACE my-internal-ca-bundle \
    --from-file=ca.crt=./corp-ca.pem
```
Once created you need to helm upgrade and restart pods for the additional CAs to be trusted.
```bash
helm upgrade $HELM_NAME . -n $NAMESPACE --reuse-values \
  --set global.privateCaBundleSecretName=my-internal-ca-bundle
```
They can be forced to restart by deleting them.
```bash
kubectl delete pod -n $NAMESPACE \
  $(kubectl get pod -n $NAMESPACE -o json | jq -r \
    '.items[] | select(.spec.volumes[]?.configMap.name == "'$HELM_NAME'-trust-cacerts") | .metadata.name')
```
The log will show the additional CAs if successfully added.
```bash
kubectl logs -n devops-system -ljob-name=main-trust --tail=500
```
The secret is not included in the normal backup scheme. You should manually backup the secret containing the additional CAs if you consider it valuable.

### Egress

In the default configuration, no egress rules are created to restrict the endpoints that the product can connect to. This enables the product to be deployed easily, without knowledge of the system under test. In environments with stricter access requirements, `networkPolicy.egress.enable` can be enabled to restrict traffic to `networkPolicy.egress.cidrs` (which defaults to private addresses defined in RFC1918). Note: With this egress policy applied, `helm test` is expected to fail due to resources used being hosted on github.com.

### Dynamic workload

To scale test asset execution the product creates kubernetes resources dynamically. To review the permissions required to do this consult the execution [role](templates/execution/role.yaml) with its [binding](templates/execution/rolebinding.yaml).

When the resources are created a label is applied so they may be tracked:
```bash
kubectl get all,cm,secret -lapp.kubernetes.io/managed-by=$HELM_NAME.$NAMESPACE -n $NAMESPACE
```
These resources are deleted 24 hours after the execution completes.

It is possible for users to request executions that exceed the resources available in the cluster. In such cases execution pods can be left Pending or Evicted. To ensure that only the dynamic workload is affected, meaning that critical services are not affected, appropriate priorityClasses need to be used within the cluster so that critical services are given priority by the scheduler.

As general guidance if your cluster has a fixed number of nodes; configure `execution.priorityClassName` with a class that has a [negative](https://kubernetes.io/blog/2019/04/16/pod-priority-and-preemption-in-kubernetes/) priority. This makes the dynamic workload the least important in the cluster thereby protecting critical services. If your cluster autoscales a negative priority can not be used since the autoscaler will not scale the cluster to meet demand from pods with a negative priority. In such cases setting a default priorityClass in the cluster with a high value for critical services is recommended with a different, lower, non-negative class for the dynamic workload using `execution.priorityClassName`. Further information can be found in the configuration section.

### Credential changes

Passwords are generated from the provided seed and stored in secrets when installing the software. These passwords can be changed in bulk by changing the seed used in the helm command, or individually by directly changing the value stored in the secret. However, for the values to become live you must run:
```bash
bash lib/migrate/reconcile-secrets.sh
```
User defined secrets used within the software are encrypted. The encryption key is also generated as above and held in a secret. It is not possible to re-encrypt these secrets without the original seed used to encrypt them. To re-encrypt the secrets, follow the steps given when running the script referenced above.

This methods should also be used when restoring a backup made where different secrets were in use.

## Limitations

* Users are required to perform backups of their data by snapshotting all persistent volume claims.

* `helm rollback` is not currently supported. Move back to a previous release by restoring a backup taken before the upgrade.
* `helm upgrade` is only supported for specific versions. See [Upgrade](#upgrade) for details.
* It is not currently possible to edit test assets. This must be done in DevOps Test Workbench.
* In each namespace, only one instance of the product can be installed.
* The replica count configuration enables a maximum of 50 active concurrent users. This configuration can not be changed.

