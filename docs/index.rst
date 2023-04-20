Introduction and Prerequisites
==============================

This hands on guide/lab will guide you on how to leverage Kcli to deploy an Hypershift cluster

General Prerequisites
---------------------

The following items are needed in order to be able to complete the lab from beginning to end:

-  A valid and powerful enough hypervisor (typically Libvirt although Vsphere, Kubevirt or oVirt can also be used).
-  A base Openshift cluster already with a default storage class available on such cluster. We will cover how to install it.
-  A valid Pull secret from `here <https://console.redhat.com/openshift/install/pull-secret>`__ that we’ll keep in a file named ‘openshift_pull.json’

About Kcli
----------

Kcli is a virtualization wrapper with support for the following providers:

-  Libvirt
-  Vsphere
-  Kubevirt
-  Aws
-  GCP
-  Ibmcloud
-  oVirt
-  Openstack
-  Packet

Additionally, it can deploy Kubernetes based cluster based on

-  Kubeadm
-  Openshift
-  Hypershift
-  Microshift
-  K3s
-  Kind

About Hypershift
----------------

Hypershift is a variant of OpenShift where the the control plane components run in a dedicated namespace of an existing base cluster, instead of spawning specific nodes for this matter.

When using Hypershift, The following objects will be created:

-  *hostedcluster* This represents the cluster and in particular the control plane components.
-  *nodepool* This represents the workers which are part of our hostedcluster. The spec of the node pool contains the following elements:
-  release image, which might be different from the one specified in the hosted cluster.
-  platform, which indicates which target platform the workers might have. Valid values can be aws, assisted, kubevirt, none,…
-  replicas, which indicates how many nodes we want to deploy.

Both objects contain a platform attribute as part of their spec, and behaviour and additional components get deployed depending on the target platform.

This lab, we will cover both ``none`` and ``assisted`` provider (as it is the supported path for bare metal deployments)

Preparing the lab
=================

Prepare the hypervisor
----------------------

**NOTE:** If you want to deploy on something else than local libvirt, refer to the `providers specific documentation <https://kcli.readthedocs.io/en/latest/#provider-specifics>`__

We install and launch Libvirt

::

   sudo dnf -y install libvirt libvirt-daemon-driver-qemu qemu-kvm
   sudo usermod -aG qemu,libvirt $(id -un)
   sudo newgrp libvirt
   systemctl enable --now libvirtd

Get Kcli
--------

We Install it with the following instructions:

::

   sudo dnf -y copr enable karmab/kcli 
   sudo dnf -y install kcli

Enable Redfish emulator
-----------------------

This step is only needed for the assisted path and allows to interact with vms created by kcli using Redfish calls.

::

   kcli create sushy-service

Create base cluster
-------------------

**NOTE:** This section can be skipped if you already have a base cluster

Now, we will deploy one cluster using a single control plane vm with dynamic storage

**NOTE:** You will typically need at least 30Gb of RAM in order for OpenShift to successfully deploy.

::

   kcli create cluster openshift -P clusterprofile=sample-openshift-sno basecluster

Expected Output

::

   Deploying on client local
   Deploying cluster basecluster
   Using stable version
   Using 192.168.122.253 as api_ip
   Setting domain to 192-168-122-253.sslip.io
   Using existing openshift-install found in your PATH
   Using installer version 4.12.13
   Using image rhcos-412.86.202303211731-0-openstack.x86_64.qcow2
   INFO Consuming Install Config from target directory
   WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
   INFO Manifests created in: /root/.kcli/clusters/basecluster/manifests and /root/.kcli/clusters/basecluster/openshift
   Forcing router pods on ctlplanes since sslip is set and api_ip will be used for ingress
   INFO Consuming Worker Machines from target directory
   INFO Consuming Master Machines from target directory
   INFO Consuming Openshift Manifests from target directory
   INFO Consuming Common Manifests from target directory
   INFO Consuming OpenShift Install (Manifests) from target directory
   INFO Ignition-Configs created in: /root/.kcli/clusters/basecluster and /root/.kcli/clusters/basecluster/auth
   Using keepalived virtual_router_id 240
   Using 192.168.122.253 for api vip....
   Deploying bootstrap
   Deploying Vms...
   Merging ignition data from existing /root/.kcli/clusters/basecluster/bootstrap.ign for basecluster-bootstrap
   basecluster-bootstrap deployed on local
   Deploying ctlplanes
   Deploying Vms...
   Merging ignition data from existing /root/.kcli/clusters/basecluster/ctlplane.ign for basecluster-ctlplane-0
   basecluster-ctlplane-0 deployed on local
   INFO Waiting up to 20m0s (until 4:45PM) for the Kubernetes API at https://api.basecluster.192-168-122-253.sslip.io:6443...
   INFO API v1.25.8+27e744f up
   INFO Waiting up to 30m0s (until 4:57PM) for bootstrapping to complete...
   INFO It is now safe to remove the bootstrap resources
   INFO Time elapsed: 17m4s
   Launching install-complete step. It will be retried one extra time in case of timeouts
   INFO Waiting up to 40m0s (until 5:22PM) for the cluster at https://api.basecluster.192-168-122-253.sslip.io:6443 to initialize...
   INFO Checking to see if there is a route at openshift-console/console...
   INFO Install complete!
   INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/.kcli/clusters/basecluster/auth/kubeconfig'
   INFO Access the OpenShift web-console here: https://console-openshift-console.apps.basecluster.192-168-122-253.sslip.io
   INFO Login to the console with user: "kubeadmin", and password: "XXXX-XXXX-XXXX-XXXX"
   INFO Time elapsed: 10m50s
   Deleting basecluster-bootstrap
   Adding app lvms-operator
   Forcing namespace to openshift-storage
   namespace/openshift-storage created
   operatorgroup.operators.coreos.com/lvms-operator-operatorgroup created
   subscription.operators.coreos.com/lvms-operator created
   Waiting for CRD LVMCluster to be created
   Waiting for CRD LVMCluster to be created
   Waiting for CRD LVMCluster to be created
   Waiting for CRD LVMCluster to be created
   lvmcluster.lvm.topolvm.io/lvmcluster created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   Waiting for the storageclass to be created
   NAME                 PROVISIONER   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
   lvms-vg1 (default)   topolvm.io    Delete          WaitForFirstConsumer   true                   3s
   storageclass.storage.k8s.io/lvms-vg1 patched (no change)
   Adding app users
   Adding dev user dev with password dev
   Adding admin user admin with password admin
   Warning: resource oauths/cluster is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by oc apply. oc apply should only be used on resources created declaratively by either oc create --save-config or oc apply. The missing annotation will be patched automatically.
   oauth.config.openshift.io/cluster configured
   secret/htpass-secret created
   Granting cluster-admin role to admin
   Warning: User 'admin' not found
   clusterrole.rbac.authorization.k8s.io/cluster-admin added: "admin"
   Creating cluster-admins group
   group.user.openshift.io/cluster-admins created
   Adding admin user admin to cluster-admins group
   group.user.openshift.io/cluster-admins added: "admin"

As per this output, we can use the cluster by setting properly the KUBECONFIG env variable

::

   export KUBECONFIG=/root/.kcli/clusters/basecluster/auth/kubeconfig

Hypershift installation
=======================

**NOTE:** In the remainder of this document, we assume KUBECONFIG is properly set.

In this section, we will cover how to install Hypershift

Launch the following command:

::

   kcli create app openshift multicluster-engine

Expected Output

::

   Adding app multicluster-engine
   namespace/multicluster-engine created
   operatorgroup.operators.coreos.com/multicluster-engine-operatorgroup created
   subscription.operators.coreos.com/multicluster-engine created
   Waiting for CRD MultiClusterEngine to be created
   Waiting for CRD MultiClusterEngine to be created
   Waiting for CRD MultiClusterEngine to be created
   Waiting for CRD MultiClusterEngine to be created
   multiclusterengine.multicluster.openshift.io/multiclusterengine created
   multiclusterengine.multicluster.openshift.io/multiclusterengine condition met
   pod/metal3-588c57c846-wgb9j condition met
   clusterimageset.hive.openshift.io/openshift-v4.13 created
   configmap/assisted-service-config created
   agentserviceconfig.agent-install.openshift.io/agent created
   secret/assisted-deployment-ssh-private-key created
   secret/assisted-deployment-pull-secret created

This performs the following tasks:

-  Craft proper namespace, operatorgroup and subscription of the multicluster engine (from the information available in the packagemanifest object).
-  Wait for MultiClusterEngine CRD.
-  Create a ``multiclusterengine`` CR with the hypershift addon enabled.
-  Define a provisioning CR to make sure metal3 operator is running.
-  Create the proper configuration for the assisted service.

**NOTE:** By default, assisted service also gets deployed, but it’s only mandatory when using the ``assisted``

We can check hypershift operator was deployed in its own namespace by running:

::

   oc get pod -n hypershift

Expected Output

::

   NAME                        READY   STATUS    RESTARTS   AGE
   operator-599cfcffc5-6gbbv   1/1     Running   0          19m
   operator-599cfcffc5-tx6dd   1/1     Running   0          19m

Cluster deployment
==================

Parameter file
~~~~~~~~~~~~~~

We prepare a parameter file named ``lab_params.yml`` providing relevant information for deployment

::

   version: stable
   tag: 4.12
   ingress_ip: 192.168.122.252
   workers: 2
   sslip: true

In this output, note the following elements:

-  ``version`` and ``tag`` allow to set specific versions, but they need to be supported by the Hypershift operator.
-  The ``sslip`` flag leverages sslip service so that no DNS entries are required to exist for deployment to succeed.
-  We specify an ``ingress_ip`` vip that will be added to the via a keepalived static pod injected when merging Ignition. An alternative and common way is to to use a loadbalancer ip via metallb on the installed cluster.

Deployment
~~~~~~~~~~

We launch the deployment with

::

   kcli create cluster hypershift --pf lab_params.yml lab

Expected Output

::

   Deploying cluster lab
   Using default class odf-storagecluster-ceph-rbd
   Using keepalived virtual_router_id 221
   Setting domain to 192-168-122-248.sslip.io
   Creating control plane assets
   namespace/clusters created
   secret/ci-hypershift-pull-secret created
   secret/ci-hypershift-ssh-key created
   hostedcluster.hypershift.openshift.io/ci-hypershift created
   Downloading openshift-install registry.ci.openshift.org/ocp/release:4.13 in current directory
   Move downloaded openshift-install somewhere in your PATH if you want to reuse it
   Using installer version 4.13.0-0.nightly-2023-04-18-005127
   Using image rhcos-413.92.202304131328-0-openstack.x86_64.qcow2
   nodepool.hypershift.openshift.io/ci-hypershift created
   Waiting before ignition data is available
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed

     0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
   100    16  100    16    0     0    290      0 --:--:-- --:--:-- --:--:--   290
     % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                    Dload  Upload   Total   Spent    Left  Speed

     0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
   100  341k    0  341k    0     0  6322k      0 --:--:-- --:--:-- --:--:-- 6322k
   Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "autoapprover" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "autoapprover" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "autoapprover" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "autoapprover" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
   cronjob.batch/ci-hypershift-autoapprover created
   Waiting for kubeconfig to be available
   # kubeconfig
   Waiting for kubeadmin*** be available
   # password
   Deploying workers
   Deploying Vms...
   Merging ignition data from existing /root/.kcli/clusters/lab/nodepool.ign for lab-worker-0
   lab-worker-0 deployed on local
   Merging ignition data from existing /root/.kcli/clusters/lab/nodepool.ign for lab-worker-1
   lab-worker-1 deployed on local
   Launching install-complete step. It will be retried extra times to handle timeouts
   level=info msg=Waiting up to 40m0s (until 1:18AM) for the cluster at https://10.19.135.112:31422 to initialize...
   W0420 00:38:23.137764 3940949 reflector.go:424] k8s.io/client-go/tools/watch/informerwatcher.go:146: failed to list *v1.ClusterVersion: Get "https://10.19.135.112:31422/apis/config.openshift.io/v1/clusterversions?fieldSelector=metadata.name%3Dversion&limit=500&resourceVersion=0": x509: certificate has expired or is not yet valid: current time 2023-04-20T00:38:23+02:00 is before 2023-04-19T22:40:48Z
   E0420 00:38:23.137848 3940949 reflector.go:140] k8s.io/client-go/tools/watch/informerwatcher.go:146: Failed to watch *v1.ClusterVersion: failed to list *v1.ClusterVersion: Get "https://10.19.135.112:31422/apis/config.openshift.io/v1/clusterversions?fieldSelector=metadata.name%3Dversion&limit=500&resourceVersion=0": x509: certificate has expired or is not yet valid: current time 2023-04-20T00:38:23+02:00 is before 2023-04-19T22:40:48Z
   (...)
   level=info msg=Checking to see if there is a route at openshift-console/console...
   level=info msg=Install complete!
   level=info msg=To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/.kcli/clusters/lab/auth/kubeconfig'
   level=info msg=Access the OpenShift web-console here: https://console-openshift-console.apps.lab.192-168-122-253.sslip.io
   level=info msg=Login to the console with user: "kubeadmin", and password: "XXXX-XXXX-XXXX-XXXX\n"
   level=info msg=Time elapsed: 12m18s

This will:

-  Create a hostedcluster object
-  Download locally openshift-install to properly evaluate the release image to use in the nodepool spec.
-  Wait for ignition server to be ready in the control plane and download worker ignition.
-  Create a nodepool object with the corresponding number of replicas and the correct release image.
-  Download rhcos image associated to the virtualization type and to the version.
-  Create vms using this rhcos image and by combining retrieved ignition with one specific to the node (allowing to inject static pods or force hostname).
-  Wait for the installation to complete using ``openshift-install wait-for install-complete`` subcommand.

Checking the control plane
~~~~~~~~~~~~~~~~~~~~~~~~~~

When using hypershift, the control planes component are hosted in a dedicated namespace, as we can see with

::

   oc get pod -n clusters-lab

Expected Output

::

   NAME                                                  READY   STATUS    RESTARTS   AGE
   catalog-operator-76d68cf889-wxc7b                     2/2     Running   0          55m
   certified-operators-catalog-686984f5cb-xgnsq          1/1     Running   0          55m
   cluster-api-c9c66b697-57c6x                           1/1     Running   0          57m
   cluster-autoscaler-9bb9cfd97-tbns7                    1/1     Running   0          56m
   cluster-image-registry-operator-77fd45fc44-wfclb      2/2     Running   0          55m
   cluster-network-operator-5b5b464b6c-pc6kj             1/1     Running   0          55m
   cluster-node-tuning-operator-d5799f99c-6dnbk          1/1     Running   0          55m
   cluster-policy-controller-5595cbc764-95wcn            1/1     Running   0          55m
   cluster-storage-operator-dd85cdf45-bcqjn              1/1     Running   0          55m
   cluster-version-operator-56c45796b9-bdgj6             1/1     Running   0          55m
   community-operators-catalog-6d47696f8-cflfb           1/1     Running   0          55m
   control-plane-operator-664cd8878b-mp95v               1/1     Running   0          57m
   csi-snapshot-controller-7d89bf444-rg87x               1/1     Running   0          54m
   csi-snapshot-controller-operator-595dcb54bc-2q9p2     1/1     Running   0          55m
   csi-snapshot-webhook-8c945d4f9-zcsq2                  1/1     Running   0          54m
   dns-operator-59487fbbdf-nd428                         1/1     Running   0          55m
   etcd-0                                                2/2     Running   0          56m
   hosted-cluster-config-operator-d67b4d585-ng2g6        1/1     Running   0          55m
   ignition-server-54d5f7785d-tr9xs                      1/1     Running   0          56m
   ingress-operator-577dbfd585-fxlbb                     2/2     Running   0          55m
   konnectivity-agent-84cd6b7875-9bqsk                   1/1     Running   0          56m
   konnectivity-server-7c94cbd9f-d8zs8                   1/1     Running   0          56m
   kube-apiserver-5c787c5f59-7hl2t                       3/3     Running   0          56m
   kube-controller-manager-7d54cdcc47-4kshc              1/1     Running   0          25m
   kube-scheduler-55c4c757f6-5gn8h                       1/1     Running   0          55m
   machine-approver-7fd7f47c5f-s7ftq                     1/1     Running   0          56m
   multus-admission-controller-6d5459f886-c286m          2/2     Running   0          30m
   oauth-openshift-5864b48666-p8bt7                      2/2     Running   0          54m
   olm-operator-74cd7b96-krnbf                           2/2     Running   0          55m
   openshift-apiserver-7679468b7f-rzrrp                  3/3     Running   0          25m
   openshift-controller-manager-796f49bf74-tdcrs         1/1     Running   0          55m
   openshift-oauth-apiserver-7f586b5c88-6ggmz            2/2     Running   0          55m
   openshift-route-controller-manager-858f67d7b5-6ss7p   1/1     Running   0          55m
   ovnkube-master-0                                      7/7     Running   0          30m
   packageserver-65cc888f58-llsb7                        2/2     Running   0          55m
   redhat-marketplace-catalog-767447c99b-dmsqj           1/1     Running   0          55m
   redhat-operators-catalog-88df6b978-5rl2n              1/1     Running   0          39m

Accessing the cluster
~~~~~~~~~~~~~~~~~~~~~

The kubeconfig corresponding to our installation gets stored in ``$HOME/.kcli/clusters/lab/auth/kubeconfig``

With this kubeconfig, we can check that the installation was successful

-  By Checking the vms

::

   kcli list vm

Expected output

::

   +---------------+--------+-----------------+----------------------------------------------------+----------+---------+
   |  Name         | Status | Ip              |  Source                                            |    Plan  | Profile |
   +---------------+--------+-----------------+----------------------------------------------------+----------+---------+
   | lab-worker-0  |  up    | 192.168.122.201 | rhcos-413.92.202304131328-0-openstack.x86_64.qcow2 | lab      |  kvirt  |
   | lab-worker-1  |  up    | 192.168.122.202 | rhcos-413.92.202304131328-0-openstack.x86_64.qcow2 | lab      |  kvirt  |
   +---------------+--------+-----------------+----------------------------------------------------+----------+---------+

-  By Checking the nodes

::

   export KUBECONFIG=$HOME/.kcli/clusters/lab/auth/kubeconfig
   oc get nodes

Expected output

::

   NAME           STATUS   ROLES    AGE     VERSION
   lab-worker-0   Ready    worker   6m37s   v1.25.7+eab9cc9
   lab-worker-1   Ready    worker   6m35s   v1.25.7+eab9cc9

-  By Checking the version of the cluster

::

   export KUBECONFIG=$HOME/.kcli/clusters/lab/auth/kubeconfig
   oc get clusterversion

Expected output

::

   NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
   version   4.12.12   True        False         7m8s    Cluster version is 4.12.12

-  By Checking the cluster operators

::

   export KUBECONFIG=$HOME/.kcli/clusters/lab/auth/kubeconfig
   oc get co

Expected output

::

   NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
   console                                    4.12.12   True        False         False      14m
   csi-snapshot-controller                    4.12.12   True        False         False      42m
   dns                                        4.12.12   True        False         False      12m
   image-registry                             4.12.12   True        False         False      12m
   ingress                                    4.12.12   True        False         False      41m
   insights                                   4.12.12   True        False         False      17m
   kube-apiserver                             4.12.12   True        False         False      42m
   kube-controller-manager                    4.12.12   True        False         False      42m
   kube-scheduler                             4.12.12   True        False         False      42m
   kube-storage-version-migrator              4.12.12   True        False         False      17m
   monitoring                                 4.12.12   True        False         False      15m
   network                                    4.12.12   True        False         False      12m
   node-tuning                                4.12.12   True        False         False      18m
   openshift-apiserver                        4.12.12   True        False         False      42m
   openshift-controller-manager               4.12.12   True        False         False      42m
   openshift-samples                          4.12.12   True        False         False      16m
   operator-lifecycle-manager                 4.12.12   True        False         False      41m
   operator-lifecycle-manager-catalog         4.12.12   True        False         False      41m
   operator-lifecycle-manager-packageserver   4.12.12   True        False         False      42m
   service-ca                                 4.12.12   True        False         False      17m
   storage                                    4.12.12   True        False         False      41m

Using assisted platform
-----------------------

(Optional) Create baremetal-like vms for assisted path
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If we dont have bare metal nodes, we can use the following will deploy 2 empty vms to emulate them.

-  Launch the following command:

::

   kcli create vm -P uefi_legacy=true -P start=false -P memory=20480 -P numcpus=16 -P disks=['{"size": 200, "interface": "sata"}'] -P nets=['{"name": "default", "mac": "aa:aa:aa:bb:bb:90"}'] -P plan=lab-assisted-vms -c 2 lab-assisted

Expected Output

::

   Deploying vm lab-assisted-0 from profile kvirt...
   lab-assisted-0 created on local
   Deploying vm lab-assisted-1 from profile kvirt...
   lab-assisted-1 created on local

-  Check the created vms

::

   kcli list vm

Expected Output

::

   +----------------+--------+----+---------+---------------+---------+
   |  Name          | Status | Ip |  Source |      Plan     | Profile |
   +----------------+--------+----+---------+---------------+---------+
   | lab-assisted-0 |  down  |    |         |      lab      |  kvirt  |
   | lab-assisted-1 |  down  |    |         |      lab      |  kvirt  |
   +----------------+--------+----+---------+---------------+---------+

.. _parameter-file-1:

Parameter file
~~~~~~~~~~~~~~

We prepare a parameter file named ``lab_assisted_params.yml`` providing relevant information for deployment

::

   assisted: true
   version: stable
   tag: 4.12
   ingress_ip: 192.168.122.251
   workers: 2
   sslip: true
   baremetal_hosts:
   - url: http://192.168.122.1:9000/redfish/v1/Systems/local/lab-assisted-0
     mac: "aa:aa:aa:bb:bb:90"
   - url: http://192.168.122.1:9000/redfish/v1/Systems/local/lab-assisted-1
     mac: "aa:aa:aa:bb:bb:91"

In this output, note the extra flags compared to the ``none`` approach:

-  ``assisted`` indicates that we will leverage the assisted platform.
-  The ``ingress_ip`` vip will be added to the nodes via a machineconfig.
-  the ``baremetal_hosts`` array which is mandatory and contains a list of hosts to be booted via Redfish. For each of them, we specify:

   -  a bmc url
   -  valid MAC address for the node so that metal3 operator can identify it.
   -  This array would also contain user/password credentials if using real bare metal nodes. Based on this information, bmh objects will be created.

**NOTE:** The flag ``mce_assisted`` can be set to True to install hypershift with assisted if missing.

.. _deployment-1:

Deployment
~~~~~~~~~~

We launch the deployment with

::

   kcli create cluster hypershift --pf lab_assisted_params.yml lab-assisted

Expected Output

::

   Deploying cluster lab-assisted
   Using default class odf-storagecluster-ceph-rbd
   Using 10.19.135.112 as management api ip
   Using keepalived virtual_router_id 248
   Setting domain to 192-168-122-251.sslip.io
   Creating control plane assets
   namespace/clusters configured
   namespace/clusters-lab created
   secret/lab-pull-secret created
   secret/lab-ssh-key created
   role.rbac.authorization.k8s.io/capi-provider-role created
   infraenv.agent-install.openshift.io/lab created
   secret/lab-pull-secret created
   secret/lab-ssh-key created
   hostedcluster.hypershift.openshift.io/lab created
   Downloading openshift-install from https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest-4.12
   Move downloaded openshift-install somewhere in your PATH if you want to reuse it
   Using installer version 4.12.12
   secret/lab-assisted-node-0 created
   baremetalhost.metal3.io/lab-assisted-node-0 created
   secret/lab-assisted-node-1 created
   baremetalhost.metal3.io/lab-assisted-node-1 created
   Waiting for 2 agents to appear
   configmap/assisted-ingress-lab created
   nodepool.hypershift.openshift.io/lab created
   Warning: would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "autoapprover" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "autoapprover" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "autoapprover" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "autoapprover" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
   cronjob.batch/lab-autoapprover created
   Waiting for kubeconfig to be available
   # kubeconfig
   Waiting for kubeadmin-password to be available
   # password
   Launching install-complete step. It will be retried extra times to handle timeouts
   INFO Waiting up to 40m0s (until 3:54PM) for the cluster at https://10.19.135.112:30155 to initialize...
   INFO Checking to see if there is a route at openshift-console/console...
   INFO Install complete!
   INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/.kcli/clusters/lab/auth/kubeconfig'
   INFO Access the OpenShift web-console here: https://console-openshift-console.apps.lab.192-168-122-251.sslip.io
   INFO Login to the console with user: "kubeadmin", and password: "XXXX-YYYYY-ZZZ-WWWW\n"
   INFO Time elapsed: 21m42s

This will:

-  Create a hostedcluster object.
-  Create an infraenv object.
-  Create baremetal host objects (BMH) using the spec from the parameter file. The vms will be booted via Redfish as would bare metal nodes.
-  Wait for the corresponding nodes to boot and register as agents.
-  Download locally openshift-install to properly evaluate the release image to use in the nodepool spec.
-  Create a nodepool object with the corresponding number of replicas and the correct release image.
-  Wait for the installation to complete using ``openshift-install wait-for install-complete`` subcommand.

In the cluster namespace, note the ``capi-provider`` pod which is in charge of assigning agents to a hosted cluster when replicas of a nodepool are indicated

Checking the assisted installer components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

During deployment, we created baremetal hosts object, which were annotated with a label ``infraenvs.agent-install.openshift.io: lab``

For instance, we can see the BMH associated to lab-node-0

::

   oc get bmh -n clusters-lab lab-assisted-node-0 -o yaml

Expected Output

::

   apiVersion: metal3.io/v1alpha1
   kind: BareMetalHost
   metadata:
     name: lab-assisted-node-1
     namespace: clusters-lab
     labels:
       infraenvs.agent-install.openshift.io: lab
     annotations:
       inspect.metal3.io: disabled
       bmac.agent-install.openshift.io/hostname: lab-assisted-node-1
       bmac.agent-install.openshift.io/role: worker
   spec:
     bmc:
       disableCertificateVerification: True
       address: redfish-virtualmedia+http://192.168.122.1:9000/redfish/v1/Systems/local/lab-assisted-1
       credentialsName: lab-assisted-node-1
     bootMACAddress: aa:aa:aa:bb:bb:91
     hardwareProfile: unknown
     online: true
     automatedCleaningMode: disabled
     bootMode: legacy

With this annotation, the corresponding nodes are booted via Redfish with an ISO that makes them available as part of the infraenv

::

   oc get agent -n clusters-lab

Expected Output

::

   NAME                                   CLUSTER          APPROVED   ROLE     STAGE
   0d711921-1afd-42e8-b2af-59b1e24d1b62   lab-assisted     true       worker   Done
   a1b35081-b4ec-4569-aff2-db0075eb8df2   lab-assisted     true       worker   Done

When the replicas number in the nodepool object gets changed, capi-provider component locates available agent and binds them as additional workers to the corresponding cluster.

Review
======

This concludes the lab !

In this lab, you have accomplished the following activities.

1. Deploy a successful Hypershift deployment leveraging Kcli tooling.
2. Deploy a successful Hypershift deployment leveraging Kcli tooling using assisted platform and vms emulating bare metal nodes
3. Learn how to check the resulting environment

Additional resources
====================

Documentation
-------------

-  https://hypershift-docs.netlify.app
-  https://docs.openshift.com/container-platform/4.12/architecture/control-plane.html#hosted-control-planes-overview_control-plane
-  https://kcli.readthedocs.io/en/latest

Cleaning the lab
----------------

::

   kcli delete cluster --yes lab
   kcli delete plan --yes lab-assisted
   kcli delete plan --yes lab-assisted-vms
   kcli delete cluster --yes basecluster
