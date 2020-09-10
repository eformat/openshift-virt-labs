Now let's bring all these settings together and actually launch some workloads!

We are going to launch this VM from a fairly large piece of YAML. Of course, this can also be done from the console, but we will explore that later. 

Let's review what this is going to do.

To begin with let's use the NFS volumes we created earlier to launch some VMs. We are going to create a machine called `centos8-server-nfs`. As you'll recall we have created a PVC called `centos8-nfs` that was created using the CDI utility with a Centos 8 base image. To connect the machine to the network we will utilize the `NetworkAttachmentDefinition` we created for the underlying host's secondary NIC. This is the `tuning-bridge-fixed` interface which refers to that bridge created previously. It's also important to remember that OpenShift 4.x uses Multus as it's default networking CNI so we also ensure Multus knows about this `NetworkAttachmentDefinition`. 

Additionally we have set the evictionStrategy to LiveMigrate so that any request to move the instance will use this method. We will explore this in more depth in a later lab, however note the `ACCESS MODES` set for this machine's PVC: 

~~~bash
$ oc get pvc
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
centos8-hostpath   Bound    pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb   29Gi       RWO            hostpath-provisioner   86m
centos8-nfs        Bound    nfs-pv1                                    10Gi       RWO,RWX        nfs                    99m
~~~

To enable Live Migration the PVC must be of the type "RWX". RWX stands for "ReadWriteMany". This means that _many_ nodes can mount the volume as _read-write_.

Finally we are using the images built in cloud-init functionality to set the *centos* user's password to *redhat*. **This means we can login directly to the instance as username *centos*, password *redhat*.**

> **NOTE**: There's a lot of YAML ahead. You can cut and paste, but spend some time reviewing the config and see if it makes sense _before_ you run it. To view outside of this environment go to [https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/rhpds/configs/centos8-nfs.yaml](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/rhpds/configs/centos8-nfs.yaml).

Ok, let's create the VM:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/centos8.0: CentOS 8
  name: centos8-server-nfs
  namespace: default
  labels:
    app: centos8-nfs
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/rhel8.2: 'true'
    vm.kubevirt.io/template: rhel8-server-small-v0.10.0
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.2
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        os.template.kubevirt.io/rhel8.2: 'true'
        vm.kubevirt.io/template: rhel8-server-small-v0.10.0
        vm.kubevirt.io/template.namespace: openshift
        vm.kubevirt.io/template.revision: '1'
        vm.kubevirt.io/template.version: v0.11.2
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: virtio
              name: disk0
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              macAddress: 'de:ad:be:ef:00:01'
              model: e1000
              name:  tuning-bridge-fixed
          rng: {}
        machine:
          type: pc-q35-rhel8.2.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: centos8-server-nfs
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
        - name: disk0
          persistentVolumeClaim:
            claimName: centos8-nfs
        - cloudInitNoCloud:
            userData: |-
                #cloud-config
                password: redhat
                chpasswd: {expire: False}
                write_files:
                  - content: |
                      # hi
                      DEVICE=eth0
                      HWADDR=de:ad:be:ef:00:01
                      ONBOOT=yes
                      TYPE=Ethernet
                      USERCTL=no
                      IPADDR=192.168.47.5
                      PREFIX=24
                      GATEWAY=192.168.47.1   
                      DNS1=150.239.16.11
                      DNS2=150.239.16.12
                    path:  /etc/sysconfig/network-scripts/ifcfg-eth0
                    permissions: '0644'
                runcmd:
                  - ifdown eth0
                  - ifup eth0
                  - systemctl restart qemu-guest-agent.service
          name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos8-server-nfs created
~~~

This starts to **schedule** the virtual machine across the available workers, which we can see by viewing the VM and VMI objects:

~~~bash
$ oc get vm
NAME                 AGE   RUNNING   VOLUME
centos8-server-nfs   5s    true

$ oc get vmi
NAME                 AGE   PHASE     IP    NODENAME
centos8-server-nfs   8s    Running         cluster-august-lhrd5-worker-6w62
~~~

> **NOTE**: A `vm` object is the definition of the virtual machine, whereas a `vmi` is an instance of that virtual machine definition.

After a few minutes the machine will report its IP:

> **NOTE**: Due to some environmental issues and an OpenStack Bug in OVN this will take around 2-3 minutes to report.

~~~bash
$ oc get vmi
NAME                 AGE    PHASE     IP                 NODENAME
centos8-server-nfs   117s   Running   192.168.47.5/24   cluster-august-lhrd5-worker-6w624
~~~

> **NOTE**: It may take a minute or two for the IP address to appear as it utilize the qemu-guest-agent which needs time to start up.

OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the `virt-launcher`:

~~~bash
$ oc get pods
NAME                                     READY   STATUS    RESTARTS   AGE
virt-launcher-centos8-server-nfs-5d8zd   1/1     Running   0          15s

$ oc describe pod/virt-launcher-centos8-server-nfs-5d8zd
Name:         virt-launcher-centos8-server-nfs-5d8zd
Namespace:    default
Priority:     0
Node:         cluster-august-lhrd5-worker-6w624/10.0.0.21
Start Time:   Fri, 24 Jul 2020 01:29:54 -0400
Labels:       flavor.template.kubevirt.io/small=true
              kubevirt.io=virt-launcher
              kubevirt.io/created-by=77a2be98-6efa-4ce8-baac-796fc9ef31ae
              os.template.kubevirt.io/rhel8.2=true
              vm.kubevirt.io/template=rhel8-server-small-v0.10.0
              vm.kubevirt.io/template.namespace=openshift
              vm.kubevirt.io/template.revision=1
              vm.kubevirt.io/template.version=v0.11.2
              workload.template.kubevirt.io/server=true
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.128.2.11"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "groot",
                    "interface": "net1",
                    "mac": "de:ad:be:ef:00:01",
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks: [{"interface":"net1","mac":"de:ad:be:ef:00:01","name":"tuning-bridge-fixed","namespace":"default"}]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.128.2.11"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "groot",
                    "interface": "net1",
                    "mac": "de:ad:be:ef:00:01",
                    "dns": {}
                }]
              kubevirt.io/domain: centos8-server-nfs
              Status:       Running
(...)
~~~

If you look into this launcher pod, you'll see that it has the same typical libvirt functionality as we've come to expect with RHV/OpenStack.

First get a shell on the conatiner running pod:

~~~bash
$ oc exec -it virt-launcher-centos8-server-nfs-5d8zd /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
~~~

And then you can run the usual virsh commands:

~~~bash
[root@centos8-server-nfs /]# virsh list --all
 Id   Name                         State
--------------------------------------------
 1    default_centos8-server-nfs   running
~~~

We can also verify the storage attachment, which should be via NFS:

~~~bash
[root@centos8-server-nfs /]# virsh domblklist 1
 Target   Source
----------------------------------------------------------------------------------------------------
 vda      /var/run/kubevirt-private/vmi-disks/disk0/disk.img
 vdb      /var/run/kubevirt-ephemeral-disks/cloud-init-data/default/centos8-server-nfs/noCloud.iso
 
[root@centos8-server-nfs /]# mount | grep nfs
192.168.47.16:/mnt/nfs/one on /run/kubevirt-private/vmi-disks/disk0 type nfs4 (rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.47.21,local_lock=none,addr=192.168.47.16)
~~~

And for networking:

~~~bash
[root@centos8-server-nfs /]# virsh domiflist 1
 Interface   Type     Source     Model   MAC
------------------------------------------------------------
 vnet0       bridge   k6t-net1   e1000   de:ad:be:ef:00:01
~~~

But let's go a little deeper:

~~~bash
[root@centos8-server-nfs /]# ip link | grep -A2 k6t-net1
5: net1@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master k6t-net1 state UP mode DEFAULT group default
    link/ether de:ad:be:c3:45:df brd ff:ff:ff:ff:ff:ff link-netnsid 0
6: k6t-net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
    link/ether de:ad:be:c3:45:df brd ff:ff:ff:ff:ff:ff
7: vnet0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master k6t-net1 state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether fe:ad:be:ef:00:01 brd ff:ff:ff:ff:ff:ff
~~~

That's showing that there's a bridge inside of the pod called "**k6t-net1**", with both the **"vnet0"** (the device attached to the VM), and the **"net1@if4"** device being how the packets get out onto the bridge on the hypervisor (more shortly):

~~~bash
[root@centos8-server-nfs /]# virsh dumpxml 1 | grep -A8 "interface type"
    <interface type='bridge'>
      <mac address='de:ad:be:ef:00:01'/>
      <source bridge='k6t-net1'/>
      <target dev='vnet0'/>
      <model type='e1000'/>
      <mtu size='1500'/>
      <alias name='ua-tuning-bridge-fixed'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x01' function='0x0'/>
    </interface>
~~~

Exit the shell before proceeding:

~~~bash
[root@rhel8-server-nfs /]# exit
exit

$
~~~

Now, how is this plugged on the underlying host? 

The key to this is the **"net1@if4"** device (it may be slightly different in your environment); this is one half of a **"veth pair"** that allows network traffic to be bridged between network namespaces, which is exactly how containers segregate their network traffic between each other on a container host. In this example the **"cnv-bridge"** is being used to connect the bridge for the virtual machine (**"k6t-net1"**) out to the bridge on the underlying host (**"br1"**), via a veth pair. The other side of the veth pair can be discovered as follows. First find the host of our virtual machine:

~~~bash
$  oc get vmi
NAME                 AGE     PHASE     IP                 NODENAME
centos8-server-nfs   4m15s   Running   192.168.47.5/24   cluster-august-lhrd5-worker-6w624
~~~

Then connect to it and track back the link - here you'll need to adjust the commands below - if your veth pair on the pod side was **"net1@if4"** then the **ifindex** in the command below will be **"4"**, if it was **"net1@if5"** then **"ifindex"** will be **"5"** and so on...

To do this we need to get to one of our workers, so first jump to the bastion:

~~~bash
$ oc debug node/cluster-august-lhrd5-worker-6w624
Starting pod/cluster-august-lhrd5-worker-6w624-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.21
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host
sh-4.4#

sh-4.4# export ifindex=4

sh-4.4# ip -o link | grep ^$ifindex: | sed -n -e 's/.*\(veth[[:alnum:]]*@if[[:digit:]]*\).*/\1/p'
veth1f161f0d@if5
~~~

Therefore, the other side of the link, in the example above is **"veth1f161f0d"**. You can then see that this is attached to **"br1"** as follows-

~~~bash
sh-4.4# ip link show dev veth1f161f0d
4: veth1f161f0d@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br1 state UP mode DEFAULT group default
    link/ether 4a:02:3f:2f:db:91 brd ff:ff:ff:ff:ff:ff link-netns 408b6bf0-12ae-4c32-9427-14b171dd2c2e
~~~

Note the "**master br1**" in the above output.

Or visually represented:

<img src="img/veth-pair.png" />

Exit the debug shell(s) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...

$
~~~

And as good habit ...

~~~bash
$ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~

Now that we have the NFS instance running, let's do the same for the **hostpath** setup we created. This is essentially the same as our NFS instance, except we reference the `rhel8-hostpath` PVC:

> **NOTE**: To view this yaml outside the lab environment go to [https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/configs/centos8-server-hostpath.yaml](https://github.com/RHFieldProductManagement/openshift-virt-labs/blob/rhpds/configs/centos8-server-hostpath.yaml)

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    kubevirt.io/latest-observed-api-version: v1alpha3
    kubevirt.io/storage-observed-api-version: v1alpha3
    name.os.template.kubevirt.io/centos8.0: CentOS 8
  name: centos8-server-hostpath
  namespace: default
  labels:
    app: centos8-server-hostpath
    flavor.template.kubevirt.io/small: 'true'
    os.template.kubevirt.io/rhel8.2: 'true'
    vm.kubevirt.io/template: rhel8-server-small-v0.10.0
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: '1'
    vm.kubevirt.io/template.version: v0.11.2
    workload.template.kubevirt.io/server: 'true'
spec:
  running: true
  template:
    metadata:
      creationTimestamp: null
      labels:
        flavor.template.kubevirt.io/small: 'true'
        os.template.kubevirt.io/rhel8.2: 'true'
        vm.kubevirt.io/template: rhel8-server-small-v0.10.0
        vm.kubevirt.io/template.namespace: openshift
        vm.kubevirt.io/template.revision: '1'
        vm.kubevirt.io/template.version: v0.11.2
        workload.template.kubevirt.io/server: 'true'
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
            - bootOrder: 1
              disk:
                bus: sata
              name: centos8-hostpath
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              macAddress: 'de:ad:be:ef:00:02'
              model: e1000
              name:  tuning-bridge-fixed
          rng: {}
        machine:
          type: pc-q35-rhel8.2.0
        resources:
          requests:
            memory: 2Gi
      evictionStrategy: LiveMigrate
      hostname: centos8-server-hostpath
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
        - name: centos8-hostpath
          persistentVolumeClaim:
            claimName: centos8-hostpath
        - cloudInitNoCloud:
            userData: |-
                #cloud-config
                password: redhat
                chpasswd: {expire: False}
                write_files:
                  - content: |
                      # hi
                      DEVICE=eth0
                      HWADDR=de:ad:be:ef:00:02
                      ONBOOT=yes
                      TYPE=Ethernet
                      USERCTL=no
                      IPADDR=192.168.47.6
                      PREFIX=24
                      GATEWAY=192.168.47.1   
                      DNS1=150.239.16.11
                      DNS2=150.239.16.12
                    path:  /etc/sysconfig/network-scripts/ifcfg-eth0
                    permissions: '0644'
                runcmd:
                  - ifdown eth0
                  - ifup eth0
                  - systemctl restart qemu-guest-agent.service
          name: cloudinitdisk
EOF
virtualmachine.kubevirt.io/centos8-server-hostpath created
~~~

As before we can see the launcher pod built and run:

~~~bash
$ oc get pods
NAME                                          READY   STATUS    RESTARTS   AGE
virt-launcher-centos8-server-hostpath-zpgwr   0/1     Running   0          4s
virt-launcher-centos8-server-nfs-5d8zd        1/1     Running   0          7m4

$  oc get vmi
NAME                      AGE     PHASE     IP                 NODENAME
centos8-server-hostpath   110s    Running   192.168.47.6/24   cluster-august-lhrd5-worker-6w624
centos8-server-nfs        8m50s   Running   192.168.47.5/24   cluster-august-lhrd5-worker-6w624

~~~

And looking deeper we can see the hostpath claim we explored earlier being utilized, note the `Mounts` section for where, inside the pod, the `rhel8-hostpath` PVC is attached, and then below the PVC name:

~~~bash
$ oc describe pod/virt-launcher-centos8-server-hostpath-zpgwr
Name:         virt-launcher-centos8-server-hostpath-zpgwr
Namespace:    default
Priority:     0
Node:         cluster-august-lhrd5-worker-6w624/10.0.0.21
Start Time:   Fri, 24 Jul 2020 01:36:54 -0400
Labels:       flavor.template.kubevirt.io/small=true
              kubevirt.io=virt-launcher
              kubevirt.io/created-by=fc48f107-a924-4214-b7af-3ef9de1de9bc
              kubevirt.io/domain=centos8-server-hostpath
              kubevirt.io/size=small
              os.template.kubevirt.io/centos8.0=true
              vm.kubevirt.io/name=centos8-server-hostpath
              workload.template.kubevirt.io/server=true
(...)
    Mounts:
      /var/run/kubevirt from virt-share-dir (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-infra from infra-ready-mount (rw)
      /var/run/kubevirt-private/vmi-disks/centos8-hostpath from centos8-hostpath (rw)
      /var/run/kubevirt/container-disks from container-disks (rw)
      /var/run/libvirt from libvirt-runtime (rw)

(...)
Volumes:
  centos8-hostpath:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  centos8-hostpath
    ReadOnly:   false
(...)
~~~

If we take a peek inside of that pod we can dig down further, you'll notice that it's largely the same output as the NFS step above, but the mount point is obviously not over NFS:

~~~bash
$ oc exec -it virt-launcher-centos8-server-hostpath-zpgwr /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

[root@centos8-server-hostpath /]#  virsh domblklist 1
 Target   Source
---------------------------------------------------------------------------------------------------------
 sda      /var/run/kubevirt-private/vmi-disks/centos8-hostpath/disk.img
 vda      /var/run/kubevirt-ephemeral-disks/cloud-init-data/default/centos8-server-hostpath/noCloud.iso
~~~

The problem here is that if you try and run the `mount` command to see how this is attached, it will only show the whole filesystem being mounted into the pod:

~~~bash
[root@centos8-server-hostpath /]# mount | grep centos8
/dev/mapper/coreos-luks-root-nocrypt on /run/kubevirt-private/vmi-disks/centos8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,prjquota)
~~~

However, we can see how this has been mapped in by Kubernetes by looking at the pod configuration on the host. First, recall which host your virtual machine is running on, and get the name of the launcher pod:

~~~bash
[root@rhel8-server-hostpath /]# exit
logout

$ oc get vmi/centos8-server-hostpath
NAME                      AGE     PHASE     IP                 NODENAME
centos8-server-hostpath   3m34s   Running   192.168.47.6/24   cluster-august-lhrd5-worker-6w624

$ oc get pods | grep centos8-server-hostpath
virt-launcher-centos8-server-hostpath-zpgwr   1/1     Running   0          3m49s
~~~

Next, we need to get the container ID from the pod:

~~~bash
$ oc describe pod virt-launcher-centos8-server-hostpath-zpgwr| awk -F// '/Container ID/ {print $2;}'
c7c2eb6a841ba1b697c9ecee8ab17de06a47ec18d807c0b6af26b8b0bd02893f
~~~

Now we can check on the worker node itself, remembering to adjust these commands for the worker that your hostpath based VM is running on, and the container ID from the above:

~~~bash
$ oc debug node/cluster-august-lhrd5-worker-6w624
Starting pod/cluster-august-lhrd5-worker-6w624-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.21
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host
sh-4.4# crictl inspect c7c2eb6a841ba1b697c9ecee8ab17de06a47ec18d807c0b6af26b8b0bd02893f | grep -A4 centos8-hostpath
        "containerPath": "/var/run/kubevirt-private/vmi-disks/centos8-hostpath",
        "hostPath": "/var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": false
--
          "destination": "/var/run/kubevirt-private/vmi-disks/centos8-hostpath",
          "type": "bind",
          "source": "/var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb",
          "options": [
            "rw",
--
        "io.kubernetes.cri-o.Volumes": "[{\"container_path\":\"/etc/hosts\",\"host_path\":\"/var/lib/kubelet/pods/43c263eb-69e0-430d-b8bc-0f9a0a57ac9f/etc-hosts\",\"readonly\":false},{\"container_path\":\"/dev/termination-log\",\"host_path\":\"/var/lib/kubelet/pods/43c263eb-69e0-430d-b8bc-0f9a0a57ac9f/containers/compute/c164551a\",\"readonly\":false},{\"container_path\":\"/var/run/kubevirt\",\"host_path\":\"/var/run/kubevirt\",\"readonly\":false},{\"container_path\":\"/var/run/libvirt\",\"host_path\":\"/var/lib/kubelet/pods/43c263eb-69e0-430d-b8bc-0f9a0a57ac9f/volumes/kubernetes.io~empty-dir/libvirt-runtime\",\"readonly\":false},{\"container_path\":\"/var/run/kubevirt-infra\",\"host_path\":\"/var/lib/kubelet/pods/43c263eb-69e0-430d-b8bc-0f9a0a57ac9f/volumes/kubernetes.io~empty-dir/infra-ready-mount\",\"readonly\":false},{\"container_path\":\"/var/run/kubevirt-ephemeral-disks\",\"host_path\":\"/var/lib/kubelet/pods/43c263eb-69e0-430d-b8bc-0f9a0a57ac9f/volumes/kubernetes.io~empty-dir/ephemeral-disks\",\"readonly\":false},{\"container_path\":\"/var/run/kubevirt/container-disks\",\"host_path\":\"/var/run/kubevirt/container-disks/fc48f107-a924-4214-b7af-3ef9de1de9bc\",\"readonly\":false},{\"container_path\":\"/var/run/kubevirt-private/vmi-disks/centos8-hostpath\",\"host_path\":\"/var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb\",\"readonly\":false}]",
        "io.kubernetes.pod.name": "virt-launcher-centos8-server-hostpath-zpgwr",
        "io.kubernetes.pod.namespace": "default",
        "io.kubernetes.pod.terminationGracePeriod": "30",
        "io.kubernetes.pod.uid": "43c263eb-69e0-430d-b8bc-0f9a0a57ac9f",
~~~

Here you can see that the container has been configured to have a `hostPath` from `/var/hpvolumes` mapped to the expected path inside of the container where the libvirt definition is pointing to.

Don't forget to exit (twice) before proceeding:

~~~bash
sh4.4# exit
exit
sh4.2# exit
exit

Removing debug pod ...
$

$ oc project default
Already on project "default" on server "https://172.30.0.1:443".
~~~
