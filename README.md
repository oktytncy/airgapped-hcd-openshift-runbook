# Mission Control & HCD Air‑Gapped Install on OpenShift

This runbook outlines the comprehensive procedure for deploying a production-ready Hyper-Converged Database (HCD) cluster within an air-gapped OpenShift environment. It utilizes a declarative approach to preemptively resolve network isolation and configuration constraints, ensuring a seamless startup without the need for manual patching or intervention.

## Environment Provisioning and Prerequisites

It is necessary to provision a suitable environment for the installation. The most suitable option for IBM environments is the OpenShift Cluster (OCP-V) - IBM, located under TechZone Certified Base Images, as shown in the screenshot below.

<p align="middle">
  <img src="images/1.png" alt="drawing" width="500"/>
</p>

**The following options were selected for this setup:**
- OpenShift Version: 4.18
- Worker Node Count: 3
- Worker Node Flavor: 8 vCPU x 32 GB - 100 GB ephemeral storage

Once the environment is ready, you can find the Bastion SSH connection and Bastion password details on the environment page. Using these credentials, you can connect to the server from your local machine.

```shell
(base) oktay.tuncay@Oktay-Tuncay ~ % ssh itzuser@api.itz-h1lus0.infra01-lb.wdc04.techzone.ibm.com -p 10022

[itzuser@itz-h1lus0-helper-1 ~]$ sudo su -
```

Click the OCP Console link on the same page, sign in to the OpenShift UI using the kube:admin user, and copy the token from the **Copy login command** section at the top of the page, as shown below.

<p align="middle">
  <img src="images/4.png" alt="drawing" width="500"/>
</p>

<p align="middle">
  <img src="images/5.png" alt="drawing" width="500"/>
</p>

Authenticate to the OpenShift Container Platform (OCP) command-line interface (CLI) with the copied token. 

**Note:** The token here is renewed at regular intervals.

```shell
[root@itz-h1lus0-helper-1 ~]# oc login --token=sha256~SEe2tu7ZIg2OiE_VX0huymYMM1uftpPXiTVQCpze_gs --server=https://api.itz-h1lus0.infra01-lb.wdc04.techzone.ibm.com:6443
Logged into "https://api.itz-h1lus0.infra01-lb.wdc04.techzone.ibm.com:6443" as "kube:admin" using the token provided.

You have access to 87 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "mission-control".
[root@itz-h1lus0-helper-1 ~]# 
```
**Check `OC`, `Helm`, `Docker` are installed on the bastion.**

Make sure that Helm v3.12+ and Docker are installed on the server.

```shell
oc version
helm version
docker version
```

```shell
[root@itz-h1lus0-helper-1 ~]# oc version
Client Version: 4.18.28
Kustomize Version: v5.4.2
Server Version: 4.18.28
Kubernetes Version: v1.31.13
```

In my case, I installed both Helm and Docker manually. When you check the Helm and Docker versions, you will see that they are installed, even if they were not present previously.

```shell
[root@itz-h1lus0-helper-1 ~]# helm version
bash: helm: command not found...

Install package 'helm' to provide command 'helm'? [N/y] y

 * Waiting in queue... 
 * Loading list of packages.... 
The following packages have to be installed:
 helm-3.19.0-1.el9.x86_64	The Kubernetes Package Manager
Proceed with changes? [N/y] y
...
...
```

```shell
[root@itz-h1lus0-helper-1 ~]# docker version
bash: docker: command not found...
Install package 'podman-docker' to provide command 'docker'? [N/y] y

 * Waiting in queue... 
 * Loading list of packages.... 
The following packages have to be installed:
...
...
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Client:       Podman Engine
Version:      5.6.0
API Version:  5.6.0
Go Version:   go1.24.6 (Red Hat 1.24.6-1.el9)
Built:        Mon Nov 10 08:54:39 2025
OS/Arch:      linux/amd64
```

It is recommended to check the storage class and available storage capacity before starting the installation.

**Check Storageclass**

```shell
[root@itz-h1lus0-helper-1 ~]# oc get storageclass
NAME                                             PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
ocs-external-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com      Delete          Immediate           true                   3h19m
ocs-external-storagecluster-cephfs               openshift-storage.cephfs.csi.ceph.com   Delete          Immediate           true                   3h19m
openshift-storage.noobaa.io                      openshift-storage.noobaa.io/obc         Delete          Immediate           false                  3h18m
```

You can also check the storage capacity in the OpenShift UI (OCP Console).

<p align="middle">
  <img src="images/6.png" alt="drawing" width="500"/>
</p>

**Check that `skopeo` and `yq` are installed on the bastion.**

```shell
skopeo --version
yq --version
```

In my case, I manually installed skopeo and yq. Verifying their versions shows that the installation was successful.

```shell
[root@itz-h1lus0-helper-1 ~]# skopeo --version
bash: skopeo: command not found...
Install package 'skopeo' to provide command 'skopeo'? [N/y] y

 * Waiting in queue... 
 * Loading list of packages.... 
...
...
```

```shell
[root@itz-h1lus0-helper-1 ~]# yq --version
bash: yq: command not found...
Install package 'yq' to provide command 'yq'? [N/y] y
...
...
yq (https://github.com/mikefarah/yq/) version v4.47.1
```

### Deploying Nexus (Namespace, PVC, Deployment, Services, and Route)

