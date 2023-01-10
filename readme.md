<!--

    Copyright (c) 2011-present Sonatype, Inc. All rights reserved.
    Includes the third-party code listed at http://links.sonatype.com/products/clm/attributions.
    "Sonatype" is a trademark of Sonatype, Inc.

-->
# Nexus IQ Server High Availability Helm Chart

This repository is intended to store a helm chart to create a cluster of Nexus IQ Server nodes.

## General Requirements
- A copy of the helm chart
- A Nexus IQ Server license that supports the High Availability (HA) feature
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) to run commands against a Kubernetes cluster
- [helm](https://helm.sh/docs/helm/) to install or upgrade the helm chart
- A PostgreSQL (10.7 or newer) database or a PostgreSQL-compatible service
- A Kubernetes cluster to run the helm chart on
- A shared file system to share files between all Nexus IQ Server pods in the cluster
- A load balancer to distribute requests between the Nexus IQ Server pods
- Optionally, install [`aws-vault`](https://github.com/99designs/aws-vault) [installed and configured](https://github.com/99designs/aws-vault/blob/master/USAGE.md#config)
  if you plan to use AWS Vault authentication, in which case prefix the aws/kubectl/helm commands below with `aws-vault exec <aws-profile> -- <command>`.

## Nice to have
- A [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) for dynamic provisioning

## Running

1. Start your Kubernetes cluster if needed
2. Open a console/terminal in the helm chart directory
3. Switch to the correct context to use your cluster if needed (e.g. `kubectl config use-context my-context`)
   1. To lookup the clusters, run `aws eks --region <aws_region> list-clusters`
   2. Import the context for the cluster into kube config: `aws eks --region <aws_region> update-kubeconfig --name <cluster_name>`
4. Install the [CSI driver](https://aws.amazon.com/blogs/security/how-to-use-aws-secrets-configuration-provider-with-kubernetes-secrets-store-csi-driver/) to enable AWS Secrets Manager access:
   1. `helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts`
   2. `helm upgrade --install --namespace kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set grpcSupportedProviders="aws" --set syncSecret.enabled=true`
   3. `kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml`
5. Install the helm chart dependencies via
   `helm dependency update .`
6. Install the helm chart via
   `helm install --namespace <namespace> <name> <overrides> .`.
where
   1. `<name>` can be any name for the helm chart
   2. `<namespace>` can be any namespace for the helm chart (can be created if desired by adding the flag
   `--create-namespace` or prior via `kubectl create namespace my-namespace`)
   3. `<overrides>` is a set of overrides for values in the helm chart (see below)
7. Expose the ingress if needed, which uses port `80` for http and port `443` for https by default

## Overrides

### License (required)

A Nexus IQ Server license that supports the HA feature must be installed either before the cluster starts or as it is
starting to allow multiple pods to start successfully.

The license file can either be passed directly
   ```
   --set-file iq_server.license=<license file>
   ```
where `<license file>` is the path to your Nexus IQ Server product license file

or via an existing secret
   ```
   --set iq_server.licenseSecret=<license secret>
   ```

or via an AWS secret arn
   ```
   --set secret.license.arn==<aws secret arn containing lifecycle license>
   ```

### Database (required)

An existing database can be configured as follows
   ```
   --set iq_server.database.hostname=<database hostname>
   --set iq_server.database.port=<database port>
   --set iq_server.database.name=<database name>
   --set iq_server.database.username=<database username>
   ```
the database password can either be passed directly
   ```
   --set iq_server.database.password=<database password>
   ```
or via an existing secret
   ```
   --set iq_server.database.passwordSecret=<database password secret>
   ```

or all of the database configuration parameters can be specified via AWS Secrets Manager arn
   ```
   --set secret.rds.arn=<aws secret arn containing host, port, name (database name,) username, and password properties>
   ```

### Shared File System (required)

By default, the helm chart will create both a [Persistent Volume (PV)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)
using [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) storage and a corresponding
[Persistent Volume Claim (PVC)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims).

However, there are various configuration options.
* If a PV is created, then it will match the configuration.
* If a PVC is created, then it will only bind to a PV that satisfies the configuration.

#### Size

The [capcity](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#capacity) or size can be set via
   ```
   --set iq_server.persistence.size=<storage size, default "1Gi">
   ```

#### Access Mode(s)

The [access mode(s)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) can be set via
   ```
   --set iq_server.persistence.accessModes[0]=<access mode, default "ReadWriteOnce">
   ```
Note that this should correspond to the type of PV being used and should either be set to `ReadWriteOnce` for a
`hostPath` PV or otherwise to `ReadWriteMany` for a `csi` or `nfs` PV.

#### Storage Class Name

The [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) name can be set via
   ```
   --set iq_server.persistence.storageClassName=<storage class name, default "">
   ```

#### Type

The [type](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes) can be
configured as follows.

Note a PV can only have one type, so if multiple are configured, then only one will be used. The priority for which type
will be selected if multiple are configured is shown below i.e. a type with a lower number in the below list will be
chosen above a type with a higher number.

1. **csi**
   ```
   --set iq_server.persistence.csi.driver=<csi driver name, default "efs.csi.aws.com">
   --set iq_server.persistence.csi.fsType=<filesystem type, default "">
   --set iq_server.persistence.csi.volumeHandle=<volume handle>
   --set iq_server.persistence.csi.volumeAttributes=<volume attributes>
   ```
2. **nfs**
   ```
   --set iq_server.persistence.nfs.server=<nfs server hostname>
   --set iq_server.persistence.nfs.path=<nfs server path, default "/">
   ```
3. **hostPath**
   ```
   --set iq_server.persistence.hostPath.path=<hostPath path, default "/mnt/iq-server">
   --set iq_server.persistence.hostPath.type=<hostPath type, default "DirectoryOrCreate">
   ```

#### Existing PV and PVC

If you have an existing PV and PVC you wish to use, then you only need to set the PVC via
   ```
   --set iq_server.persistence.existingPersistentVolumeClaimName=<existing persistent volume claim name>
   ```

#### Existing PV

If you have an existing PV you wish to use, then you can set the PV via
   ```
   --set iq_server.persistence.existingPersistentVolumeName=<existing persistent volume name>
   ```
However, you may need to configure the PVC that will be created to allow it to bind to the PV using the previously
mentioned configuration options.


### Load Balancer (required)

An [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) can be enabled with a particular [class
name](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) to use an existing load balancer
as follows
   ```
   --set ingress.enabled=<true|false, default false>
   --set ingress.ingressClassName=<ingress class name, default "nginx">
   --set ingress.pathType=<ingress path type, default "Prefix">
   --set ingress.hostApplicationPath=<application path, default iq_server.config.server.applicationContextPath>
   --set ingress.hostAdminPath=<admin path, default iq_server.config.server.adminContextPath>
   --set ingress.hostApplication=<application hostname>
   --set ingress.hostAdmin=<admin hostname>
   --set ingress.annotations=<ingress annotations>
   ```
Note that if you want both the application and admin endpoints to be accessible, then they will need to be set to have
either different hostnames via e.g.
   ```
   --set iq_server.config.server.hostApplication="app.domain"
   --set iq_server.config.server.hostAdmin="admin.domain"
   ```
or different paths via e.g.
   ```
   --set iq_server.config.server.applicationContextPath="/app"
   --set iq_server.config.server.adminContextPath="/admin"
   ```

Note that if no hostnames are specified, then any web traffic to the IP address of your ingress controller can be
matched without a name based virtual host being required.

#### TLS

If your [ingress class](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class) supports 
specifying TLS options directly based on your [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/),
then you can specify them as follows.

A TLS certificate and private key can either be passed directly
   ```
   --set ingress.tls[0].certificate=<tls certificate file>
   --set ingress.tls[0].key=<tls private key file>
   ```
where `<tls certificate file>` is the path to your TLS certificate file and `<tls private key file>` is the path to your
TLS private key file, or via an existing secret
   ```
   --set ingress.tls[0].secretName=<tls secret name>
   ```
The TLS secret must contain keys named `tls.cert` for the TLS certificate and `tls.key` for the TLS private key.

Additionally multiple hosts can be specified as follows
   ```
   --set ingress.tls[0].hosts[0]=<tls hostname>
   ```

Alternatively some ingress classes may support specifying TLS options through annotations.

### Nexus IQ Server Configuration (optional)

The number of pods can be specified as follows
   ```
   --set iq_server.replicas=<number of pods, default 2>
   ```

The initial admin password can either be passed directly
   ```
   --set iq_server.initialAdminPassword=<initial admin password>
   ```
or via an existing secret
   ```
   --set iq_server.initialAdminPasswordSecret=<initial admin password secret>
   ```
or via an existing secret
   ```
   --set secret.arn==<aws secret arn containing initial admin password in lifecycle_admin_password property>
   ```

A `config.yml` file is required to run. This is generated using the `iq_server.config` value. Care should be taken if
updating this as many values within it are fine-tuned to allow the helm chart to function.

### fluentd (required)

fluentd is required to generate aggregated log files which are needed for any support requests.

An aggregator is used to write aggregated log files to the PV.

The aggregator receives logs from forwarders.

By default, each forwarder is a sidecar container that runs alongside the Nexus IQ Server container in the same pod.
This forwards the Nexus IQ Server log file lines, which it has access to through a shared file system, to the
aggregator.

Note that fluentd has separate settings for its PVC and should normally be configured to use the same PVC as the Nexus
IQ Server pods as follows
   ```
   --set fluentd.aggregator.extraVolumes[0].name="iq-server-pod-volume"
   --set fluentd.aggregator.extraVolumes[0].persistentVolumeClaim.claimName=<PVC name, default "iq-server-pvc">
   ```

### Image (optional)

By default, the
[latest publicly available Nexus IQ Server docker image](https://hub.docker.com/r/sonatype/nexus-iq-server)
will be used.

The image, tag, and imagePullPolicy can be overridden using
   ```
   --set iq_server.image=<image>
   --set iq_server.tag=<tag>
   --set iq_server.imagePullPolicy=<imagePullPolicy>
   ```

## Amazon Web Services (AWS)

### Satisfying General Requirements
- [Relational Database Service (RDS) for PostgreSQL](https://aws.amazon.com/rds/) for a PostgreSQL database
- [Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/) for a cluster
- [Elastic File System (EFS)](https://aws.amazon.com/efs/) with mount targets for a shared file system
- [Application Load Balancer (ALB)](https://aws.amazon.com/elasticloadbalancing/application-load-balancer/) for a load
balancer

### Additional Requirements
- [Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) for the AWS
resources to communicate with each other
- [Amazon EFS CSI driver](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html) pre-installed and configured in
the cluster

### Nice to have
- [EFS Storage Class](https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml)
pre-installed and configured in the cluster for [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [AWS Load Balancer Controller add-on](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
pre-installed and configured in the cluster to automatically provision an ALB based on an ingress
- [AWS CloudWatch](https://aws.amazon.com/cloudwatch/) configuration for fluentd to send aggregated logs to

### EFS

An existing EFS drive with mount targets can be used for the PV.

The PV can be provisioned statically or dynamically. In either case, the access modes should be set as follows
   ```
   --set iq_server.persistence.accessModes[0]="ReadWriteMany"
   ```

#### Static Provisioning

To statically provision the PV use the following

   ```
   --set iq_server.persistence.csi.volumeHandle=<EFS file system ID>[:<EFS file system path>]
   ```
where `<EFS file system ID>` is your EFS file system ID, which typically looks like e.g. "fs-0ac8d13f38bfc99df" and
`<EFS file system path>` is an optional path into the file system e.g. ":/".

#### Dynamic Provisioning

To dynamically provision the PV via an EFS storage class use the following
   ```
   --set iq_server.persistence.persistentVolumeName=""
   --set iq_server.persistence.storageClassName=<EFS storage class name>
   ```

### ALB

For an ALB load balancer to work you will need to change the Nexus IQ Server [service type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
from its default of `ClusterIP` to `NodePort` via the following
   ```
   --set iq_server.serviceType=NodePort
   ```

#### Dynamic Provisioning

To dynamically provision an ALB via the AWS Load Balancer Controller add-on use the following
   ```
   --set ingress.enabled=true
   --set ingress.ingressClassName=alb
   ```
and ensure that the [appropriate annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/)
are set e.g.
   ```
   --set ingress.annotations."external-dns\.alpha\.kubernetes\.io/hostname"="domain"
   --set ingress.hostApplication="domain"
   --set ingress.hostAdmin="admin.domain"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"="internet-facing"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/target-type"="ip"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/healthcheck-path"="/ping"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/certificate-arn"="arn:aws:acm:us-east-2:119982741446:certificate/8a0a2dbf-6283-48f2-b636-4faacb0665c9"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/ssl-policy"="ELBSecurityPolicy-FS-1-2-Res-2020-10"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/listen-ports"='\[\{\"HTTPS\":80\}\,\{\"HTTPS\":443\}\]'
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/actions\.ssl-redirect"='\{\"Type\": \"redirect\"\,\"RedirectConfig\":\{\"Protocol\":\"HTTPS\"\,\"Port\":\"443\"\,\"StatusCode\":\"HTTP_301\"\}\}'
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/actions\.response-404"='\{\"type\":\"fixed-response\"\,\"fixedResponseConfig\":\{\"contentType\":\"text/plain\"\,\"statusCode\":\"404\"\,\"messageBody\":\"404_Not_Found\"\}\}'
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/actions\.redirect-domain"='\{\"Type\":\"redirect\"\,\"RedirectConfig\":\{\"Host\":\"domain\"\,\"Path\":\"/#\{path\}\"\,\"Port\":\"443\"\,\"Protocol\":\"HTTPS\"\,\"Query\":\"#\{query\}\"\,\"StatusCode\":\"HTTP_301\"\}\}'
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/load-balancer-attributes"="idle_timeout.timeout_seconds=600"
   ```

### EFS Storage Class

If you want to use dynamic provisioning, then an EFS storage class should be pre-installed and configured in the cluster
with the correct settings to allow read/write access to the Nexus IQ Server pod users, which have UID 1000 and GID 1000
by default e.g.
   ```
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
     metadata:
     name: efs-sc
   provisioner: efs.csi.aws.com
   parameters:
     provisioningMode: efs-ap
     fileSystemId: <EFS file system ID>
     directoryPerms: "777"
     gidRangeStart: "1000"
     gidRangeEnd: "1000"
     basePath: "/"
   ```

### CloudWatch fluentd

The fluentd aggregator can be configured to send aggregated logs to CloudWatch using the
[fluent-plugin-cloudwatch-logs plugin](https://github.com/fluent-plugins-nursery/fluent-plugin-cloudwatch-logs).

This requires fluentd aggregator pods to have the [correct permissions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-prerequisites.html)
, which can either be associated with a service account the fluentd aggregator uses, or with the EKS worker nodes.

Once the permissions are established, you can enable sending aggregated logs to CloudWatch, set the AWS region,
log group name, and log stream name as follows
   ```
   --set cloudwatch.enabled=true
   --set cloudwatch.region=<AWS region>
   --set cloudwatch.logGroupName=<CloudWatch log group name>
   --set cloudwatch.logStreamName=<CloudWatch log stream name>
   ```

### Examples

Some example commands are shown below.

#### External Database, Static EFS, and Dynamic ALB
   ```
   helm install mycluster
   --set-file iq_server.license="license.lic"
   --set iq_server.database.hostname=myhost
   --set iq_server.database.port=5432
   --set iq_server.database.name=iq
   --set iq_server.database.username=postgres
   --set iq_server.database.password=admin123
   --set iq_server.config.server.applicationContextPath="/app"
   --set iq_server.config.server.adminContextPath="/admin"
   --set iq_server.persistence.accessModes[0]="ReadWriteMany"
   --set iq_server.persistence.csi.volumeHandle="fs-0ac8d13f38bfc99df:/"
   --set iq_server.serviceType=NodePort
   --set ingress.enabled=true
   --set ingress.ingressClassName=alb
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"="internet-facing"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/healthcheck-path"="/app/ping" 
   .
   ```

#### External Database, Dynamic EFS, and Dynamic ALB
   ```
   helm install mycluster
   --set-file iq_server.license="license.lic"
   --set iq_server.database.hostname=myhost
   --set iq_server.database.port=5432
   --set iq_server.database.name=iq
   --set iq_server.database.username=postgres
   --set iq_server.database.password=admin123
   --set iq_server.config.server.applicationContextPath="/app"
   --set iq_server.config.server.adminContextPath="/admin"
   --set iq_server.persistence.accessModes[0]="ReadWriteMany"
   --set iq_server.persistence.persistentVolumeName=""
   --set iq_server.persistence.storageClassName="efs-storage-class-name"
   --set iq_server.serviceType=NodePort
   --set ingress.enabled=true
   --set ingress.ingressClassName=alb
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/scheme"="internet-facing"
   --set ingress.annotations."alb\.ingress\.kubernetes\.io/healthcheck-path"="/app/ping" 
   .
   ```

## On-Premises

### Satisfying General Requirements
* Any PostgreSQL database, we recommend one [setup for HA](https://www.postgresql.org/docs/current/high-availability.html)
* Any Kubernetes cluster, we recommend a multi-node cluster [setup for HA](https://kubernetes.io/docs/setup/production-environment/)
* Any shared file system, we recommend a [Network File System (NFS)](https://en.wikipedia.org/wiki/Network_File_System)
* Any [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) pre-installed
and configured in the cluster, we recommend the [ingress-nginx controller](https://github.com/kubernetes/ingress-nginx)

### Example

An example command is shown below.

#### External Database, NFS, and ingress-nginx
   ```
   helm install mycluster
   --set-file iq_server.license="license.lic"
   --set iq_server.database.hostname=myhost
   --set iq_server.database.port=5432
   --set iq_server.database.name=iq
   --set iq_server.database.username=postgres
   --set iq_server.database.password=admin123
   --set iq_server.config.server.applicationContextPath="/app"
   --set iq_server.config.server.adminContextPath="/admin"
   --set iq_server.persistence.accessModes[0]="ReadWriteMany"
   --set iq_server.persistence.nfs.server=10.109.77.85
   --set iq_server.persistence.nfs.path=/
   --set iq_server.serviceType=NodePort
   --set ingress.enabled=true
   --set ingress-nginx.enabled=true
   .
   ```

## Upgrading
In order to upgrade Nexus IQ Server and ensure a successful data migration, the following steps are recommended:
1. **Scale your pods down to zero.** This will delete the existing pods.
2. **Backup the database.** See the [IQ server backup guidelines](https://links.sonatype.com/products/nxiq/doc/backup) for more details.
3. **Run your helm chart upgrade command.** The deleted pods will be re-created with the updates.
