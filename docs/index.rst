Introduction and Prerequisites
==============================

This hands on lab will provide you instructions so you can deploy an Hypershift cluster. The goal is to make you understand Hypershift internals and workflow so that you can understand the workflow and value proposition.

Although other platform types can be used for deploying an Hypershift cluster (such as Kubevirt or None, for which Kcli tool actually has support), we will focus on the assisted installer approach, since it’s the recommended way to deploy bare metal workers.

About Hypershift
----------------

HyperShift is middleware for hosting OpenShift control planes at scale that solves for cost and time to provision, as well as portability cross cloud with strong separation of concerns between management and workloads.

Clusters are fully compliant OpenShift Container Platform (OCP) clusters and are compatible with standard OCP and Kubernetes toolchains.

When using Hypershift, we will deploy OpenShift on top of an existing cluster by creating the following objects:

-  *hostedcluster* This represents the cluster and in particular the control plane components that will be “hosted” in a dedicated cluster
-  *nodepool* This represents the workers part of our hostedcluster. Its spec contains the following elements:

   -  release image, which might be different from the one specified in the hosted cluster.
   -  platform, which indicates which target platform the workers might have. Valid values can be aws, kubevirt, none, …
   -  replicas, which indicates how many nodes we want to deploy

Both objects contain a platform attribute as part of their spec, and behaviour and additional components get deployed depending on the target platform. For instance, if we use Kubevirt, additional components will be deployed so that vms get deployed depending on the replicas spec of the nodepool.

In this lab, we are using the assisted provider, suitable for bare metal deployments. It relies on assisted service being already deployed, and an *infraenv* exists in target cluster.

General Prerequisites
---------------------

The following items are needed in order to be able to complete the lab from beginning to end:

-  A base Openshift cluster already running and a KUBECONFIG pointing to it
-  A default storage class available on such cluster
-  A valid and powerful enough hypervisor (typically Libvirt, although Vsphere, oVirt can be used too)
-  A valid Pull secret from `here <https://console.redhat.com/openshift/install/pull-secret>`__ that we’ll keep in a file named ‘openshift_pull.json’
-  `Kcli <https://kcli.readthedocs.io/en/latest>`__ tool to ease some of the tasks.
-  Oc binary in your path (this can be downloaded using ``kcli download oc``

**NOTE:** You will need around 50Gb of RAM in order for OpenShift to successfully deploy. If those requirements are not met, you can still run but the final deployment might not succeed.

Preparing the lab
=================

**NOTE:** This section can be skipped if lab has been prepared for you.

Prepare the hypervisor
----------------------

**NOTE:** If you want to deploy on something else than local libvirt, refer to Kcli documentation

We install and launch libvirt

::

   sudo dnf -y install libvirt libvirt-daemon-driver-qemu qemu-kvm
   systemctl enable --now libvirtd

Get Kcli
--------

We will leverage `Kcli <https://kcli.readthedocs.io/en/latest>`__ to easily create the assets needed for the lab.

Install it with the following instructions:

::

   sudo dnf -y copr enable karmab/kcli 
   sudo dnf -y install kcli

Create baremetal-like vms
-------------------------

-  Launch the following command:

::

   kcli create vm -P uefi=true -P start=false -P memory=20480 -P numcpus=16 -P disks=['{"size": 200, "interface": "sata"}'] -P nets=['{"name": "default", "mac": "aa:aa:aa:bb:bb:90"}'] -c 3 -P plan=lab-vms lab

Expected Output

::

   Deploying vm lab-0 from profile kvirt...
   lab-0 created on local
   Deploying vm lab-1 from profile kvirt...
   lab-1 created on local
   Deploying vm lab-2 from profile kvirt...
   lab-2 created on local

This will deploy 3 empty ctlplane vms to emulate bare metal nodes.

-  Check the created vms

::

   kcli list vm

Expected Output

::

   +------------------------+--------+-----------------+------------------------------------------------------+---------------+---------+
   |          Name          | Status |        Ip       |                        Source                        |      Plan     | Profile |
   +------------------------+--------+-----------------+------------------------------------------------------+---------------+---------+
   |         lab-0          |  down  |                 |                                                      |      lab      |  kvirt  |
   |         lab-1          |  down  |                 |                                                      |      lab      |  kvirt  |
   |         lab-2          |  down  |                 |                                                      |      lab      |  kvirt  |
   +------------------------+--------+-----------------+------------------------------------------------------+---------------+---------+

Hypershift installation
=======================

In this section, we install Hypershift

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

-  Craft proper namespace, operatorgroup and subscription of the multicluster engine from package manifest
-  Wait for MultiClusterEngine CRD
-  Create a multiclusterengine CR with the hypershift add on enabled
-  Define a provisioning CR to make sure metal3 operator is running
-  Create the proper configuration for the assisted service

We can check the pods deployed by using

::

   oc get pod -n multicluster-engine

Expected Output

::

   multicluster-engine                                agentinstalladmission-78464dd777-bdwgf                                1/1     Running            0               11m
   multicluster-engine                                agentinstalladmission-78464dd777-fr7rt                                1/1     Running            0               11m
   multicluster-engine                                assisted-image-service-0                                              1/1     Running            0               11m
   multicluster-engine                                assisted-service-6769dff9b9-cng9b                                     2/2     Running            0               11m
   multicluster-engine                                cluster-curator-controller-55976b8d7d-dzc2j                           1/1     Running            0               13m
   multicluster-engine                                cluster-curator-controller-55976b8d7d-stf6x                           1/1     Running            0               13m
   multicluster-engine                                cluster-image-set-controller-6447fc7b6d-tksb9                         1/1     Running            0               13m
   multicluster-engine                                cluster-manager-65b886b48-8hz4v                                       1/1     Running            0               13m
   multicluster-engine                                cluster-manager-65b886b48-8z5fq                                       1/1     Running            0               13m
   multicluster-engine                                cluster-manager-65b886b48-sg98x                                       1/1     Running            0               13m
   multicluster-engine                                cluster-proxy-addon-manager-6b8575dc55-cljxd                          1/1     Running            0               12m
   multicluster-engine                                cluster-proxy-addon-manager-6b8575dc55-g78wg                          1/1     Running            0               12m
   multicluster-engine                                cluster-proxy-addon-user-8c9cb664b-78bvd                              2/2     Running            0               12m
   multicluster-engine                                cluster-proxy-addon-user-8c9cb664b-pndlg                              2/2     Running            0               12m
   multicluster-engine                                cluster-proxy-c6f9ff875-9fqlt                                         1/1     Running            0               12m
   multicluster-engine                                cluster-proxy-c6f9ff875-kdr74                                         1/1     Running            0               12m
   multicluster-engine                                clusterclaims-controller-66b6748d7d-n9vsp                             2/2     Running            0               13m
   multicluster-engine                                clusterclaims-controller-66b6748d7d-tmwhq                             2/2     Running            0               13m
   multicluster-engine                                clusterlifecycle-state-metrics-v2-6c64ddf44b-59xx6                    1/1     Running            0               13m
   multicluster-engine                                console-mce-console-5f4886bd56-lhkmm                                  1/1     Running            0               13m
   multicluster-engine                                console-mce-console-5f4886bd56-plpr7                                  1/1     Running            0               13m
   multicluster-engine                                discovery-operator-86d4f65f76-ks8ml                                   1/1     Running            0               13m
   multicluster-engine                                hive-operator-6667956b88-plqvm                                        1/1     Running            0               13m
   multicluster-engine                                hypershift-addon-manager-78f84b794c-ggssq                             1/1     Running            0               13m
   multicluster-engine                                hypershift-cli-download-6695fcf9c-hwwh8                               1/1     Running            0               12m
   multicluster-engine                                infrastructure-operator-5d88f5677f-2rxrk                              1/1     Running            0               13m
   multicluster-engine                                managedcluster-import-controller-v2-6f556c9555-j8f6v                  1/1     Running            0               13m
   multicluster-engine                                managedcluster-import-controller-v2-6f556c9555-s867f                  1/1     Running            0               13m
   multicluster-engine                                multicluster-engine-operator-bbf4f7645-btv24                          1/1     Running            0               13m
   multicluster-engine                                multicluster-engine-operator-bbf4f7645-q6rm7                          1/1     Running            0               13m
   multicluster-engine                                ocm-controller-689c99d59c-55xmh                                       1/1     Running            0               13m
   multicluster-engine                                ocm-controller-689c99d59c-xmxbl                                       1/1     Running            0               13m
   multicluster-engine                                ocm-proxyserver-6f4f7d487-l9rl4                                       1/1     Running            0               13m
   multicluster-engine                                ocm-proxyserver-6f4f7d487-xrb2n                                       1/1     Running            0               13m
   multicluster-engine                                ocm-webhook-769f6c7f7d-6ct8h                                          1/1     Running            0               13m
   multicluster-engine                                ocm-webhook-769f6c7f7d-pgswn                                          1/1     Running            0               13m
   multicluster-engine                                provider-credential-controller-77647dbcdc-4zftp                       2/2     Running            0               13m

Hypershift operator was also deployed in its own namespace

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
==============

We prepare a parameter file named ``lab_params.yml`` providing relevant information for deployment

::

   version: stable
   tag: 4.12
   ingress_ip: 192.168.122.248
   workers: 3
   sslip: true
   baremetal_hosts:
   - url: http://192.168.122.1:9000/redfish/v1/Systems/local/lab-0
     mac: "aa:aa:aa:bb:bb:90"
   - url: http://192.168.122.1:9000/redfish/v1/Systems/local/lab-1
     mac: "aa:aa:aa:bb:bb:91"
   - url: http://192.168.122.1:9000/redfish/v1/Systems/local/lab-2
     mac: "aa:aa:aa:bb:bb:92"

In this output, note

-  version and tag allow to set specific versions, as long as they are supported by the Hypershift operator
-  we specify an ingress vip that will be added to the nodes via machineconfig. An alternative and common way would be to use a loadbalancer ip via metallb
-  the baremetal_hosts array contains a list of hosts to be booted via Redfish. For each of them , we specify a bmc url and a valid MAC address for the node so that metal3 operator can identify it. This array would also contain user/password credentials if using real bare metal nodes.

**NOTE:** The boolean flag can be set to assisted to install hypershift and assisted if missing.

We launch the deployment with

::

   kcli create cluster hypershift --pf lab_params.yml lab

Expected Output

::

   Deploying cluster lab
   Using default class odf-storagecluster-ceph-rbd
   Using 10.19.135.112 as management api ip
   Using keepalived virtual_router_id 248
   Setting domain to 192-168-122-248.sslip.io
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
   secret/lab-node-0 created
   baremetalhost.metal3.io/lab-node-0 created
   secret/lab-node-1 created
   baremetalhost.metal3.io/lab-node-1 created
   secret/lab-node-2 created
   baremetalhost.metal3.io/lab-node-2 created
   Waiting for 3 agents to appear
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
   INFO Access the OpenShift web-console here: https://console-openshift-console.apps.lab.192-168-122-248.sslip.io
   INFO Login to the console with user: "kubeadmin", and password: "XXXX-YYYYY-ZZZ-WWWW\n"
   INFO Time elapsed: 21m42s

This will:

-  Create a hostedcluster object
-  Create an infraenv object
-  Create baremetal host objects (bmh) using the spec from the parameter file. The vms will be booted via redfish as would bare metal nodes.
-  Wait for the corresponding nodes to boot and register as agents
-  Download locally openshift-install to properly evaluate the release image to use in the nodepool spec
-  Create a nodepool object with the corresponding number of replicas and the correct release image
-  Wait for the installation to complete using openshift-install ``wait-for install-complete`` subcommand

Checking the control plane
==========================

When using hypershift, the control planes component are hosted in a dedicated namespace, as we can see with

::

   oc get pod -n clusters-lab

Expected Output

::

   NAME                                                  READY   STATUS    RESTARTS   AGE
   capi-provider-67c67c9c4f-vvxgh                        1/1     Running   0          57m
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

In this output, note the ``capi-provider`` pod which is in charge of assigning agents to a hosted cluster when replicas of a nodepool are indicated

Checking the assisted installer components
==========================================

During deployment, we created baremetal hosts object, which were annotated with a label ``infraenvs.agent-install.openshift.io: lab``

For instance, we can see it the bmh associated to lab-node-0

::

   oc get bmh -n clusters-lab lab-node-0 -o yaml

Expected Output

::

   apiVersion: metal3.io/v1alpha1
   kind: BareMetalHost
   metadata:
     name: lab-node-1
     namespace: clusters-lab
     labels:
       infraenvs.agent-install.openshift.io: lab
     annotations:
       inspect.metal3.io: disabled
       bmac.agent-install.openshift.io/hostname: lab-node-1
       bmac.agent-install.openshift.io/role: worker
   spec:
     bmc:
       disableCertificateVerification: True
       address: redfish-virtualmedia+http://192.168.122.1:9000/redfish/v1/Systems/local/lab-1
       credentialsName: lab-node-1
     bootMACAddress: aa:aa:aa:bb:bb:91
     hardwareProfile: unknown
     online: true
     automatedCleaningMode: disabled
     bootMode: legacy

With this annotation, the corresponding nodes are booted via Redfish with an iso that makes them available as part of the infraenv

::

   oc get agent -n clusters-lab

Expected Output

::

   NAME                                   CLUSTER   APPROVED   ROLE     STAGE
   0d711921-1afd-42e8-b2af-59b1e24d1b62   lab       true       worker   Done
   a1b35081-b4ec-4569-aff2-db0075eb8df2   lab       true       worker   Done
   e26fc0e1-9fd4-42c2-8959-7ff9acb8fe8f   lab       true       worker   Done

When the replicas number in the nodepool object gets changed, capi-provider component tries to locate available agent and plug them as additional workers to the corresponding cluster

Accessing the cluster
=====================

The kubeconfig corresponding to our installation gets stored in ``$HOME/.kcli/clusters/lab/auth/kubeconfig`` but we can also retrieve it manually using the following command:

::

   CLUTSTER=lab
   oc extract -n clusters secret/$CLUSTER-admin-kubeconfig --to=- > kubeconfig.$CLUSTER

Expected output

::

   # kubeconfig

With the kubeconfig, we can check how the installation is successful

-  By Checking the nodes

::

   export KUBECONFIG=$HOME/.kcli/clusters/lab/auth/kubeconfig
   oc get nodes

Expected output

::

   NAME         STATUS   ROLES    AGE     VERSION
   lab-node-0   Ready    worker   6m37s   v1.25.7+eab9cc9
   lab-node-1   Ready    worker   6m35s   v1.25.7+eab9cc9
   lab-node-2   Ready    worker   5m35s   v1.25.7+eab9cc9

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

Review
======

This concludes the lab !

In this lab, you have accomplished the following activities.

1. Properly prepare a successful Hypershift deployment leveraging Assisted operator and Kcli tool.
2. Deploy OpenShift on vms the same way you would deploy on bare metal Nodes !
3. Understand internal aspects of the workflow and which specific objects are key when leveraging Hypershift

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

   kcli delete cluster hypershift lab
   kcli delete plan --yes lab-vms
