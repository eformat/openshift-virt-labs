In this section, we configure the networking for our environment. 

With OpenShift Virtualization we have a few different options for networking - we can just have our virtual machines be attached to the same pod networks that our containers would have access to, or we can configure additioanl real-world virtualization networking constructs like bridged networking, SR/IOV, and so on. It's also absolutely possible to have a combination of these, e.g. both pod networking and a bridged interface directly attached to a VM at the same time. This is achieved via Multus, the default Container Network Interface (CNI) in OpenShift 4.x.

In this lab we're going to enable multiple options - pod networking and a secondary network interface provided by a bridge on the underlying worker nodes (hypervisors). Each of the worker nodes has been configured with an additional, currently unused, network interface. To utilize this, we will need a linux bridge device, `br2`, to be created so we can attach our virtual machines to it. 

The first step is to use the new Kubernetes NetworkManager state configuration to setup the underlying hosts to our liking. Recall that we can get the **current** state by requesting the `NetworkNodeState`:

~~~bash
$ oc get nodes

NAME                                   STATUS   ROLES    AGE   VERSION
master-0.lab01.redhatpartnertech.net   Ready    master   9h    v1.18.3+6c42de8
master-1.lab01.redhatpartnertech.net   Ready    master   9h    v1.18.3+6c42de8
master-2.lab01.redhatpartnertech.net   Ready    master   9h    v1.18.3+6c42de8
worker-0.lab01.redhatpartnertech.net   Ready    worker   9h    v1.18.3+6c42de8
worker-1.lab01.redhatpartnertech.net   Ready    worker   9h    v1.18.3+6c42de8
worker-2.lab01.redhatpartnertech.net   Ready    worker   9h    v1.18.3+6c42de8
~~~

~~~bash
$ oc get nns/worker-0.lab01.redhatpartnertech.net -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkState
metadata:
  creationTimestamp: "2020-09-10T00:04:00Z"
  generation: 1
  name: worker-0.lab01.redhatpartnertech.net
  ownerReferences:
  - apiVersion: v1
    kind: Node
    name: worker-0.lab01.redhatpartnertech.net
    uid: 1b184f11-6574-46ed-b309-9fe5e4cb628d
  resourceVersion: "299066"
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
    interfaces:
    - ipv4:
        enabled: false
      ipv6:
        enabled: false
      mac-address: 7e:fc:fb:cd:74:44
      mtu: 1450
      name: br0
      state: down
      type: ovs-interface
(...)
~~~

In there you'll spot the interface that we'd like to use to create a bridge, `eno2`, ignore the IP address that it has right now, that came from the DHCP server on the lb/bastion node.

~~~bash
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
        address:
        - ip: fe80::f117:9d3f:f9b9:331d
          prefix-length: 64
        auto-dns: true
        auto-gateway: true
        auto-routes: true
        autoconf: true
        dhcp: true
        enabled: true
      mac-address: E4:43:4B:DE:0F:E1
      mtu: 1500
      name: eno2
      state: up
      type: ethernet
~~~

Now let's create a policy to create a bridge on the nodes using this NIC. We want this to be on all workers, so we need to create a nodeSelector that ensure every worker is selected. We do that with `node-role.kubernetes.io/worker: ""` This applies the config to all worker nodes.

However, be sure you're using the right NIC naming, as it *may* be different in your environment to the examples shown below. You can easily use the Node Network State to find this, for instance in the example above you can check the available NICs like this:

~~~bash
$ oc get nns/worker-0.lab01.redhatpartnertech.net -o yaml | grep eno | grep name
      name: eno1
      name: eno2
      name: eno3
      name: eno4
      
$ oc get nns/worker-1.lab01.redhatpartnertech.net -o yaml | grep eno | grep name
      name: eno1
      name: eno2
      name: eno3
      name: eno4

$ oc get nns/worker-2.lab01.redhatpartnertech.net -o yaml | grep eno | grep name
      name: eno1
      name: eno2
      name: eno3
      name: eno4
~~~

**With the above output we can see the NICs on the worker nodes are eno1 through eno4. The systems in the deployed cluster are configured with Packet Cloud's Mixed/Hybrid network mode. The first NIC on the server (named eth0 in Packet admin console, named eno1 by the RHEL CoreOS operating system) is part of an LACP bond - with the second NIC from the bond (named eth1 in Packet admin console, named eno2 by the RHEL CoreOS operating system) broken out and put in L2 mode on a VLAN within the Packet data center.  If your NICs are different, change the YAML below to reflect that.**

~~~bash
$ cat << EOF > br2-eno2-worker-nncp.yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br2-eno2-policy-workers
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br2
        type: linux-bridge
        state: up
        bridge:
          options:
            group-forward-mask: 0
            mac-ageing-time: 300
            multicast-snooping: true
            stp:
              enabled: true
              forward-delay: 15
              hello-time: 2
              max-age: 20
              priority: 32768
          port:
            - name: eno2 <---- CHANGE HERE AND ABOVE IF DIFFERENT NIC
              stp-hairpin-mode: false
              stp-path-cost: 100
              stp-priority: 32
EOF

$ vi br2-eno2-worker-nncp.yaml
(change the port name entry if your system doesn't have eno2)

$ oc apply -f br2-eno2-worker-nncp.yaml
nodenetworkconfigurationpolicy.nmstate.io/br2-eno2-policy-workers created
~~~

Then check as to whether it was successfully applied:

~~~bash
$ oc get nncp
NAME                      STATUS
br2-eno2-policy-workers   SuccessfullyConfigured

$ oc get nnce
NAME                                                           STATUS
master-0.lab01.redhatpartnertech.net.br2-eno2-policy-workers   NodeSelectorNotMatching
master-1.lab01.redhatpartnertech.net.br2-eno2-policy-workers   NodeSelectorNotMatching
master-2.lab01.redhatpartnertech.net.br2-eno2-policy-workers   NodeSelectorNotMatching
worker-0.lab01.redhatpartnertech.net.br2-eno2-policy-workers   SuccessfullyConfigured
worker-1.lab01.redhatpartnertech.net.br2-eno2-policy-workers   SuccessfullyConfigured
worker-2.lab01.redhatpartnertech.net.br2-eno2-policy-workers   SuccessfullyConfigured
~~~

> **NOTE**: Only our workers will have matched the node selector, hence why only two of the above nodes show as "SuccessfullyConfigured".

We can also dive into the `NetworkNodeConfigurationPolicy` (**nncp**) a little further:

~~~bash
$ oc get nncp/br2-eno2-policy-workers -o yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"nmstate.io/v1alpha1","kind":"NodeNetworkConfigurationPolicy","metadata":{"annotations":{},"name":"br2-eno2-policy-workers"},"spec":{"desiredState":{"interfaces":[{"bridge":{"options":{"group-forward-mask":0,"mac-ageing-time":300,"multicast-snooping":true,"stp":{"enabled":true,"forward-delay":15,"hello-time":2,"max-age":20,"priority":32768}},"port":[{"name":"eno2","stp-hairpin-mode":false,"stp-path-cost":100,"stp-priority":32}]},"name":"br2","state":"up","type":"linux-bridge"}]},"nodeSelector":{"node-role.kubernetes.io/worker":""}}}
  creationTimestamp: "2020-09-10T06:37:03Z"
  generation: 1
(...)
  name: br2-eno2-policy-workers
  resourceVersion: "327015"
  selfLink: /apis/nmstate.io/v1alpha1/nodenetworkconfigurationpolicies/br2-eno2-policy-workers
  uid: c71318d2-7b17-42ee-8d82-fd2767043d20
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          group-forward-mask: 0
          mac-ageing-time: 300
          multicast-snooping: true
          stp:
            enabled: true
            forward-delay: 15
            hello-time: 2
            max-age: 20
            priority: 32768
        port:
        - name: eno2
          stp-hairpin-mode: false
          stp-path-cost: 100
          stp-priority: 32
      name: br2
      state: up
      type: linux-bridge
  nodeSelector:
    node-role.kubernetes.io/worker: ""
status:
  conditions:
  - lastHearbeatTime: "2020-09-10T06:37:13Z"
    lastTransitionTime: "2020-09-10T06:37:12Z"
    reason: SuccessfullyConfigured
    status: "False"
    type: Degraded
  - lastHearbeatTime: "2020-09-10T06:37:13Z"
    lastTransitionTime: "2020-09-10T06:37:12Z"
    message: 3/3 nodes successfully configured
    reason: SuccessfullyConfigured
    status: "True"
    type: Available
~~~
 
Now that the "physical" networking is configured on the underlying worker nodes, we need to then define a `NetworkAttachmentDefinition` so that when we want to use this bridge, OpenShift and OpenShift Virtualization know how to attach to it. This associates the bridge we just defined with a logical name, known here as '**tuning-bridge-fixed**':

> **NOTE**: Since the name for the bridge `br2` was applied to all three worker nodes in our br2-eno2-policy-workers NodeNetworkConfigurationPolicy, we don't need to create multiple versions of this `NetworkAttachmentDefinition`.

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: tuning-bridge-fixed
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br2
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "groot",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br2"
      },
      {
        "type": "tuning"
      }
    ]
  }'
EOF

networkattachmentdefinition.k8s.cni.cncf.io/tuning-bridge-fixed created
~~~

> **NOTE**: The important flags to recognise here are the **type**, being **cnv-bridge** which is a specific implementation that links in-VM interfaces to a counterpart on the underlying host for full-passthrough of networking. Also note that there is no **ipam** listed - we don't want the CNI to manage the network address allocation for us - the network we want to attach to has DHCP enabled, and so let's not get involved.

When needing to query NetworkAttachmentDefinition(s), the shortcut name is "net-attach-def":

~~~bash
$ oc get net-attach-def
NAME                  AGE
tuning-bridge-fixed   2m4s
