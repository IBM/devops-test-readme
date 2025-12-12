# HCL DevOps Test Hub

HCL DevOps Test Hub brings together test data, test environments, and test runs and reports into a single, web-based browser for testers and non-testers. It is Kubernetes native and hence requires a cluster to run. If you do not have a cluster available we enable you to get started by providing scripts to provision a basic environment using K3s.

## Prerequisites


The product requires a minimum of: (in addition to resources required by the cluster)
* 16GiB memory
* 8 cpu
* 128GiB of persistent storage

Depending on workload considerably more resources could be required.

To install the product you will need cluster administrator privileges.


## Install

Fetch chart for install:
```bash
helm pull --untar oci://hclcr.io/ot/hcl-devops --version 11.0.6
cd hcl-devops
```


### Ingress controller


Install Emissary
```bash
bash lib/ingress/main.sh

kubectl rollout status -n emissary deployment/emissary-ingress -w

# wait until the external-ip is assigned after a few minutes
kubectl get svc -n emissary -w emissary-ingress
INGRESS_IP= # external-ipv4 address
```

### Chart
```bash
NAMESPACE=devops-system
HELM_NAME=main

INGRESS_DOMAIN=$INGRESS_IP.nip.io
PASSWORD_SEED= # secure seed required to generate passwords - unrecoverable so keep it safe

HCL_LICENSING_URL=
HCL_LICENSING_ID=

helm upgrade --install $HELM_NAME . -n $NAMESPACE \
  --create-namespace \
  --set global.domain=$INGRESS_DOMAIN \
  -f values-k8s.yaml \
  --set global.persistence.rwxStorageClass=rook-ceph-file \
  --set-literal passwordSeed=$PASSWORD_SEED \
  --set signup=true \
  --set hclLicensingURL=$HCL_LICENSING_URL \
  --set hclLicensingID=$HCL_LICENSING_ID
```
* When the ingress domain is accessible to untrusted parties, `signup` must be set to `false`.
* The password seed is used to generate default passwords and should be stored securely. Its required again to restore from a backup.

* The rwxStorageClass is cloud provider dependent, the value provided is only an example.


### Configuration

| Parameter                                      | Description | Default |
|------------------------------------------------|-------------|---------|
| `execution.ingress.hostPattern`                | Pattern used to generate hostnames so that running assets may be accessed via ingress. | PLATFORM specifc |
| `execution.nodePorts.enabled`                  | When `network.policy` is disabled, allow NodePorts to be used to access to running assets like virtual services. | true |
| `global.domain`                                | The web address to expose the product on. For example `192.168.0.100.nip.io` | REQUIRED |
| `global.hclCertSecretName`                     | Optionally used to terminate TLS and when `ingress.cert.selfSigned`, is used to verify trust of loopback connections. | ingress |
| `global.hclImagePullSecret`                    | The docker-registry secret to pull images from the `imageRegistry`. | '' |
| `global.hclImagePullUsername`                  | Username to pull images from the `imageRegistry`. | 'cp' |
| `global.hclImagePullPassword`                  | Password to pull images from the `imageRegistry`. | '' |
| `global.persistence.rwoStorageClass`           | The storageClass to use if the cluster default is not appropriate. | '' |
| `global.persistence.rwxStorageClass`           | For environments that do not provide a default StorageClass that supports the ReadWriteMany (RWX) accessMode, this value must be set to a suitable StorageClass that supports ReadWriteMany access. | REQUIRED |
| `global.privateCaBundleSecretName`             | Name of secret containing `ca.crt`, which lists certificates to trust (in PEM format). | '' |
| `hclLicensingID`                                 | The license server id, also known as the deployment id. For example `abc11def` | REQUIRED |
| `hclLicensingURL`                                | The location of either the internal license server or the default external cloud license server. | REQUIRED |
| `imageRegistry`                                | The location of container images to use. See [move-images](lib/airgap/move-images.sh) | hclcr.io/ot |
| `ingress.cert.create`                          | Create an self-signed certificate matching the ingress domain if none exists in secret `global.hclCertSecretName`. | true |
| `ingress.cert.selfSigned`                      | If the ingress domain certificate is not signed by a globally trusted CA. | PLATFORM specifc |
| `keycloak.truststoreFileHostnameVerificationPolicy` | HTTPS hostname cerificate verifcation policy. ANY (hostname is not verified), WILDCARD (allows wildcards in subdomain names) or STRICT (the Common Name (CN) must match the hostname exactly). | WILDCARD |
| `networkPolicy.egress.cidrs`                   | Network ranges to allow access to. This does not include access to github.com where helm test resources are stored. | [ 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 ] |
| `networkPolicy.egress.enable`                  | When `network.policy` is enabled create a rule to narrow egress from the product. | false |
| `networkPolicy.enabled`                        | Deny other software, installed in the cluster, access to the product. | true |
| `passwordSeed`                                 | The seed used to generate all passwords. | REQUIRED |
| `postgresql.migrate.enabled`                   | Enable Postgresql version migration on start when coming from v10.5.3. Migration is disabled to avoid an unnecessary image pull. | false |
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


#### [S3](https://github.com/vmware-tanzu/velero-plugin-for-aws)

Prepare the cluster to use CSI Snapshots by installing the [CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter?tab=readme-ov-file#usage).

Provision S3 storage with your cloud provider. As an example we'll use [MinIO](https://min.io/).

To use velero with S3 storage, the AWS storage plugin must be used with a `credentials` file in the following form:

```text
[default]
aws_access_key_id = ACCESS_KEY_ID
aws_secret_access_key = SECRET_ACCESS_KEY
```

Install velero using the AWS plugin and CSI feature:

```bash
S3_IP=127.0.0.1
REGION=minio
BUCKET_NAME=$NAMESPACE
velero install \
    --provider aws \
    --features=EnableCSI \
    --plugins=velero/velero-plugin-for-aws:v1.10.1 \
    --bucket "$BUCKET_NAME" \
    --secret-file ./credentials \
    --snapshot-location-config region=$REGION \
    --backup-location-config region=$REGION,s3ForcePathStyle="true",s3Url=https://${S3_IP}:9000 \
    --use-node-agent \
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


## Security Considerations

### Ingress

#### Firewall

The product loops back some requests via the ingress controller. It this is blocked by a firewall some pods will fail to transition to Running without it.

#### Trust of generated self signed certificate

When necessary the product generates a CA and certificate to terminate TLS. To fetch the generated CA so that it can be injected into other softwares trust stores, see the notes from:
```bash
helm status $HELM_NAME -n $NAMESPACE
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

* `helm rollback` is not currently supported. Move back to a previous release by restoring a backup taken before the upgrade.
* `helm upgrade` is only supported for specific versions. See [Upgrade](#upgrade) for details.
* It is not currently possible to edit test assets. This must be done in DevOps Test Workbench.
* In each namespace, only one instance of the product can be installed.
* The replica count configuration enables a maximum of 50 active concurrent users. This configuration can not be changed.

