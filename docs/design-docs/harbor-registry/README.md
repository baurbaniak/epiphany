# Docker Registry implementation design document 
# Goals
Provide Docker container registry as a Epiphany service. Registry  for application containers storage, docker image signs and docker image security scanning. 
# Use cases
Store application docker images in private registry.
Sign docker images with passphrase to be trusted.
Automated security scanning of docker images which are pushed to the registry.

# Architectural decision


## Comparison of the available solutions

Considered options:

 - Harbor [https://github.com/goharbor/](https://github.com/goharbor/)
 - Quay [https://github.com/quay/](https://github.com/quay/)
 - Portus [https://github.com/SUSE/Portus](https://github.com/SUSE/Portus)

Feature comparison table 

|Feature  | Harbor | Quay.io | Portus	
|--|--|--|--|
|Ability to Determine Version of Binaries in Container  |Yes |Yes |Yes |
|Audit Logs | Yes | Yes | Yes |
|Content Trust and Validation|Yes | Yes | Yes |
|Custom TLS Certificates|Yes|Yes|Yes|
|Helm Chart Repository Manager|Yes|Partial|Yes|
|Open source|Yes|Partial|Yes|
|Project Quotas (by image count & storage consumption)|Yes|No|No|
|Replication between instances|Yes|Yes|Yes|
|Replication between non-instances|Yes|Yes|No|
|Robot Accounts for Helm Charts|Yes|No|Yes|
|Robot Accounts for Images|Yes|Yes|Yes|
|Tag Retention Policy|Yes|Partial|No|
|Vulnerability Scanning & Monitoring|Yes|Yes|Yes|
|Vulnerability Scanning Plugin Framework|Yes|Yes|No|
|Vulnerability Whitelisting|Yes|No|No|
|Complexity of the installation process|Easy|Difficult|Difficult|
|Complexity of the upgrade process|Medium|Difficult|Difficult|

Source of comparison: [https://goharbor.io/docs/1.10/build-customize-contribute/registry-landscape/](https://goharbor.io/docs/1.10/build-customize-contribute/registry-landscape/)
and also based on own experience (stack installation and upgrade).

# Design proposal

## Harbor services architecture 

![enter image description here](https://raw.githubusercontent.com/goharbor/harbor/master/docs/1.10/img/harbor-architecture-1.10.png)

## Implementation architecture
Additional components are required for Harbor implementation.  

- Shared storage volume between kubernetes nodes ( NFS),
- Component for TLS/SSL certificate request (maybe cert-manager?), 
- Component for TLS/SSL certificate store and manage certificate validation (maybe Vault?), 
- Component for TLS/SSL certificate share between server and client (maybe Vault?), 
- HELM component for deployment procedure,
- Load balancer (HAProxy) for access to Portal and Notary service.


Diagram for TLS certificate management:


![enter image description here](./CertDiagram.png)



Kubernetes deployment diagram:

![enter image description here](./KubernetesDiagram.png)


## Implementation steps
- Deploy shared storage service (in example NFS) for K8s cluster (M/L)
- Deploy Helm3 package manager and also Helm Charts for offline installation (S/M)
- Deploy Hashicorp Vault for self-signed PKI for Harbor (external task + S for Harbor configuration)
- Deploy "cert request/management" service and integrate with Hashicorp Vault - require research (M/L)
- Deploy Harbor services using Helm3 with self-signed TLS certs (for non-production environments) (L)
- Deploy Harbor services using Helm3 with commercial TLS certs (for prod environments) (M/L)
- Load balancer (HAProxy) setup.

## Step by step procedure for deploy Demo environment

1. Deploy Epiphany cluster with components: NFS server, Kubernetes cluster (master node and at least one node), load balancer (HAProxy) and PostgreSQL database.

2. Setup HAProxy config file:

```
frontend https_front
    mode tcp
    option forwardfor
    option forwardfor header X-Real-IP
    http-request set-header X-Forwarded-Proto https
    bind *:443 ssl crt /etc/ssl/haproxy/self-signed-test.tld.pem
    default_backend http_back1
backend http_back1
   balance roundrobin
   http-request add-header X-Forwarded-Proto https
   server node1 {KUBERNETES_PORTAL_SERVICE_IP}:30002 check
frontend https_notary
    mode tcp
    option forwardfor
    option forwardfor header X-Real-IP
    http-request set-header X-Forwarded-Proto https
    bind *:4443 ssl crt /etc/ssl/haproxy/self-signed-test.tld.pem
    default_backend http_back_notary
backend http_back_notary
    balance roundrobin
    http-request add-header X-Forwarded-Proto https
    server node1 {KUBERNETES_NOATRY_SERVICE_IP}:30004 check
```
3. Setup persistent volumes on Kubernetes. Login to Master node and create PV and PVC.

harbor-nfs-persistent-volume.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: harbor-nfs-volume
  labels:
    name: harbor-nfs-volume
spec:
  storageClassName: defaultfs
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  mountOptions:
    - hard
    - nfsvers=4.1
    - rsize=1048576
    - wsize=1048576
    - timeo=600
    - retrans=2
  nfs:
    path: /mnt/nfs_share
    server: {nfs_server_ip_address}
```
harbor-nfs-persistent-volume-claim.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: harbor-nfs-volume-claim
spec:
  storageClassName: defaultfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      name: harbor-nfs-volume
```      
4. Manually create databases in PostgreSQL

```
create database harbor;
create database clair;
create database notary_signer;
create database notary_server;
create user harbor with encrypted password 'Harbor12345';
grant all privileges on database harbor to harbor;
grant all privileges on database clair to harbor;
grant all privileges on database notary_signer to harbor;
grant all privileges on database notary_server to harbor;
```
4. Login to Kubernetes Master node and add Harbor Helm official repository:

```
helm repo add harbor https://helm.goharbor.io
```

5. Deploy Harbor from helm repository with parameters:

```
helm install release-name harbor/harbor --set expose.type=nodePort \
--set expose.tls.enabled=false \
--set persistence.persistentVolumeClaim.registry.existingClaim=harbor-nfs-volume-claim \
--set persistence.persistentVolumeClaim.registry.accessMode=ReadWriteMany \
--set persistence.persistentVolumeClaim.registry.subPath=/data \
--set persistence.persistentVolumeClaim.chartmuseum.existingClaim=harbor-nfs-volume-claim \
--set persistence.persistentVolumeClaim.chartmuseum.accessMode=ReadWriteMany \
--set persistence.persistentVolumeClaim.jobservice.existingClaim=harbor-nfs-volume-claim \
--set persistence.persistentVolumeClaim.jobservice.accessMode=ReadWriteMany \
--set persistence.persistentVolumeClaim.redis.existingClaim=harbor-nfs-volume-claim \
--set persistence.persistentVolumeClaim.redis.accessMode=ReadWriteMany \
--set persistence.persistentVolumeClaim.trivy.existingClaim=harbor-nfs-volume-claim \
--set persistence.persistentVolumeClaim.trivy.accessMode=ReadWriteMany \
--set imagePullPolicy=Always \
--set database.type=external \  
--set database.external.host={POSTGRES_SERVER_IP} \
--set database.external.username=harbor \
--set database.external.password=Harbor12345 \ 
--set trivy.enabled=false \
--set externalURL={LOAD_BALACER_PUBLIC_IP}
```



