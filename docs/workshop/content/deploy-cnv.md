**OpenShift Virtualization** is available as both upstream (**KubeVirt**) and downstream releases. As of this writing the downstream release is version 2.4 and is available from the OperatorHub. **This version is fully supported by Red Hat.** 

The mechanism for installation is to utilize the operator model and deploy via the OpenShift Operator Hub (Marketplace) in the web-console. It's entirely possible to deploy via the CLI should you wish to do so, but we're not documenting that mechanism here.

Use the **Console** portion of the lab for the next steps. As a reminder, the access is via the buttons at the top. Any CLI commands should be executed by clicking back to the **Terminal** via the same buttons.

<img  border="1" src="img/console-button.png"/>

Next, navigate to the top-level '**Operators**' menu entry, and select '**OperatorHub**'. This lists all of the available operators that you can install from the Red Hat Marketplace. Simply start typing '**virtualization**' in the search box and you should see an entry called "OpenShift Virtualization". 

<img  border="1" src="img/ocp-virt-operator-panel-3.png"/>

Select it and you'll see a window that looks like the following:

<img  border="1" src="img/ocp-virt-operator-install-3.png"/>

Click the 'Install' button. This takes you to a second window where you can set some details about the installation. Spend some time to review it, but you can leave the defaults as they'll automatically select the latest version of OpenShift Virtualization and will allow the software to be installed automatically:

<img  border="1" src="img/ocp-virt-operator-install-details-2.png"/>

Make sure that the namespace it will be installed to is "**openshift-cnv**" - it should be the default entry, but make sure. When you're ready, press the **'Install'** button. The install process progress will be displayed:

<img  border="1" src="img/ocp-virt-operatore-install-progress.png"/>

Wait for the install to complete successfully; it may take a few minutes. Once Status reaches Succeeded, you can proceed:

<img  border="1" src="img/ocp-virt-operatore-install-success.png"/>

Next we need to deploy the rest of the OpenShift Virtualization components that this operator provides. Make sure to be on the "**Installed Operators** pane, then select the "**OpenShift Virtualization**" link under the '**Name**' column:

<img  border="1" src="img/ocp-virt-operator-deploy.png"/>

...and you will be presented with the following:

<img  border="1" src="img/ocp-virt-hco-1.png"/>

From here, select '**+ Create Instance**' on the '**CNV Operator Deployment**' mini pane. You'll be presented with the '**Create HyperConverged Cluster**' screen. This will deploy all of the necessary components that are required to support OpenShift Virtualization. Just go with the defaults and click the '**Create**' button.

<img  border="1" src="img/ocp-virt-hco-2.png"/>

The '**kubevirt-hyperconverged**' operator will begin to install.

<img  border="1" src="img/ocp-virt-hco-3.png"/>

You can follow the install by moving to the **Workloads** -> **Pods** menu on the left. Choose **Filter** and select **Pending** to watch the pods deploy.

<img src="img/ocp-virt-hco-4.png"/>

You can also watch this via the CLI:

~~~bash
$ watch -n2 'oc get pods -n openshift-cnv'
(...)
~~~

> **NOTE**: Press **Ctrl+C** when ready to exit the watch command.

During this process you will see a lot of pods create and terminate, which will look something like the following depending on when you view it; it's always changing:

<img src="img/deploy-cnv-watch2.png"/>

This will continue for some time, depending on your environment. Please ensure it completes successfully.

You will know the process is complete when you can return to the terminal and see that the operator installation has been successful by running the following command:

~~~bash
$ oc get csv -n openshift-cnv
NAME                                      DISPLAY                    VERSION   REPLACES   PHASE
kubevirt-hyperconverged-operator.v2.4.1   OpenShift Virtualization   2.4.1                Succeeded
~~~

If you do not see `Succeeded` in the `PHASE` column then the deployment may still be progressing, or has failed. You will not be able to proceed until the installation has been successful. Once the `PHASE` changes to `Succeeded` you can validate that the required resources and the additional components have been deployed across the nodes. First let's check the pods deployed in the `openshift-cnv` namespace:

~~~bash
$ oc get pods -n openshift-cnv
NAME                                                 READY   STATUS    RESTARTS   AGE
bridge-marker-h9hgl                                  1/1     Running   0          13m
bridge-marker-j76lr                                  1/1     Running   0          13m
bridge-marker-ljjgr                                  1/1     Running   0          13m
bridge-marker-rf8vj                                  1/1     Running   0          13m
bridge-marker-zxp52                                  1/1     Running   0          13m
cdi-apiserver-7b5894bdbb-77p28                       1/1     Running   0          13m
cdi-deployment-b4f97d69f-ncp22                       1/1     Running   0          13m
cdi-operator-5f9b9c977b-xw7w2                        1/1     Running   0          14m
cdi-uploadproxy-76c94b65c-x25dt                      1/1     Running   0          13m
(...)
~~~

> **NOTE**: All pods shown from this command should be in the `Running` state. You will have many different types, the above snippet is just an example of the output at one point in time, you may have more or less at any one point. Below we discuss some of the pod types and what they do.


Together, all of these pods are responsible for various functions of running a virtual machine on-top of OpenShift/Kubernetes. See the table below that describes some of the various different pod types and their function:

| Pod Name                             | Pod Responsibilities |
| ------------------------------------ | -------------------- |
| *[bridge-marker](https://github.com/kubevirt/bridge-marker)*                      | Marks network bridges as available node resources.|
| *[cdi-*](https://github.com/kubevirt/containerized-data-importer)*                              |  The Containerised Data Importer (CDI) is a Kubernetes extension to populate PVCs with VM disk images or other data. CDI pods allow OpenShift Virtualization to import, upload and clone Virtual Machine images. |
| *[cluster-network-addons-operator](https://github.com/kubevirt/cluster-network-addons-operator)*    | Allows the installation of additional networking plugins. |
| *[hco-operator](https://github.com/kubevirt/hyperconverged-cluster-operator)*                       | Allows users to deploy and configure multiple operators in a single operator and via a single entry point. An "operator of operators." |
| *[hostpath-provisioner-operator](https://github.com/kubevirt/hostpath-provisioner-operator)*      |Operator that manages the hostpath-provisioner, which provisions storage on network filesystems mounted on the host.|
| *[kube-cni-linux-bridge-plugin](https://github.com/containernetworking/plugins)*       |CNI Plugin to create a network bridge and add a host and container to it.|
| *kubemacpool-mac-controller-manager* |Allocation of MAC addresses from a pool to secondary interfaces.|
| *[kubevirt-node-labeller](https://github.com/kubevirt/node-labeller)*             |Creates node labels based on CPU information.|
| *[kubevirt-ssp-operator](https://github.com/MarSik/kubevirt-ssp-operator)*              |Scheduling, Scale and Performance operator for OpenShift. The Hyperconverged Cluster Operator automatically installs the SSP operator when deploying.|
| *nmstate-handler*                    |Deploys NMState which allows network administrators to manage host networking settings in a declarative manner.|
| *[node-maintenance-operator](https://github.com/kubevirt/cluster-network-addons-operator#nmstate)*|Operator that allows the administrator to deploy the NMState State Controller.                    |
| *[ovs-cni](https://github.com/kubevirt/ovs-cni)*|The Open vSwitch CNI plugin.|
| *[virt-api](https://github.com/kubevirt/kubevirt/tree/master/pkg/virt-api)*                           |Kubernetes Virtualization API and runtime in order to define and manage virtual machines|
| *[virt-controller](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)*                    |The operator that’s responsible for cluster-wide virtualization functionality|
| *[virt-handler](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)*                       |Tracks changes to a VM's state.|
| *[virt-template-validator](https://kubernetes.io/blog/2018/05/22/getting-to-know-kubevirt/)*            |Add-on to check the annotations on templates and reject them if invalid.|


There's also a few custom resources that get defined too, for example the `NodeNetworkState` (`nns` for short) definitions that can be used with the `nmstate-handler` pods to ensure that the NetworkManager state on each node is configured as required. This is used for easily defining interfaces/bridges on each of the machines allowing connectivity for both the physical machine itself and for providing network access for pods (and virtual machines) within OpenShift/Kubernetes:

~~~bash
$ oc get nns -A
NAME                                   AGE
master-0.lab01.redhatpartnertech.net   4m16s
master-1.lab01.redhatpartnertech.net   4m17s
master-2.lab01.redhatpartnertech.net   4m16s
worker-0.lab01.redhatpartnertech.net   4m17s
worker-1.lab01.redhatpartnertech.net   4m17s
worker-2.lab01.redhatpartnertech.net   4m17s
~~~

Dive a little deeper into the config on one of the workers (some fields removed for brevity):

~~~bash
[~] $ oc get nns/worker-0.lab01.redhatpartnertech.net -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2020-09-10T00:04:00Z"
  generation: 1
    manager: kubernetes-nmstate
    operation: Update
    time: "2020-09-10T00:09:11Z"
  name: worker-0.lab01.redhatpartnertech.net
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: worker-0.lab01.redhatpartnertech.net
    uid: 1b184f11-6574-46ed-b309-9fe5e4cb628d
  resourceVersion: "111125"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkstates/worker-0.lab01.redhatpartnertech.net
  uid: 6a854d88-9aef-4f09-b238-2384d4d6731f
status:
  currentState:
    dns-resolver:
      config:
        search: []
        server: []
      running:
        search:
        - example.local.dc
        server:
        - 147.75.207.207
        - 147.75.207.208
        - 8.8.8.8
        - 8.8.4.4
    interfaces:
    - ipv4:
        enabled: false
      ipv6:
        enabled: false
      mac-address: 4e:49:a0:5d:47:45
      mtu: 1450
      name: br0
      state: down
      type: ovs-interface
    - ethernet:
        auto-negotiation: false
        duplex: full
        speed: 10000
      ipv4:
        address:
        - ip: 136.144.55.157
          prefix-length: 31
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:
        address:
        - ip: fe80::7b63:be82:5fca:fc2f
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      mac-address: E4:43:4B:DE:0F:E0
      mtu: 1500
      name: eno1
      state: up
      type: ethernet
    - ethernet:
        auto-negotiation: false
        duplex: full
        speed: 10000
      ipv4:
        address:
        - ip: 192.168.2.104
          prefix-length: 24
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        dhcp: true
        enabled: true
      ipv6:

~~~

Here you can see the current state of the node (some of the output has been cut), the interfaces attached, and their physical/logical addresses. In a later section we're going to be modifying the network node state by applying a new configuration to allow nodes to utilize another interface to provide pod networking via a **bridge**. We will do this via a `NodeNetworkConfigurationEnactment` or `nnce` in short:

~~~bash
$ oc get nnce -n openshift-cnv
No resources found in openshift-cnv namespace.
~~~

As we've not set any additional configuration at this stage, it's perfectly normal to have 'no resources found' in the output above. We are going to build this in a later lab!


### Viewing the OpenShift Virtualization Dashboard

When OpenShift Virtualization is deployed it adds additional components to OpenShift's web-console so you can interact with objects and custom resources defined by OpenShift Virtualization, including `VirtualMachine` types. If you select the `Console` button at the top of this pane you should see the web-console displayed. You can now navigate to "**Workloads**" --> "**Virtualization**" on the left-hand side panel and you should see the new component for OpenShift Virtualization. Of course there aren't any Virtual Machines running yet.

<img src="img/ocpvirt-dashboard.png"/>

**Please don't try and create any virtual machines just yet, we'll get to that shortly. First, we need to set up Storage and Networking...**
