In this lab we're going to clone a workload and prove that it's identical to the source. For fun, we will use a Centos 7 image for this work. We're going to download and customise the image, launch a virtual machine via OpenShift Virtualization based on it, and then clone it - we'll then test to see if the cloned machine works as expected. 

Let's begin by checking we have a available PV for this work:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                             5h20m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                             5h19m
pvc-9346e93a-4c9f-4266-9b00-4b1a329aed1d   100Gi      RWX            Delete           Bound       openshift-image-registry/image-registry-storage   managed-nfs-storage             12h
pvc-9462894f-cbf5-455f-8465-709ec6d6f428   222Gi      RWO            Delete           Bound       default/centos8-hostpath                          hostpath-provisioner            3h57m
~~~

In this case, PV nfs-pv1 is marked as `Available`. (Your environment could have nfs-pv1 bound and nfs-pv2 available instead)

Next we will create a PVC for that PV that utilizes the CDI utility to import a Centos 7 image (from http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2). The syntax should be familiar to you now with the `cdi.kubevirt.io/storage.import.endpoint` annotation indicating the endpoint for CDI to import from.

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos7-clone-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/centos7-clone-nfs created
~~~

As before we can watch the process and see the pods:

~~~bash
$ oc get pods
NAME                                          READY   STATUS    RESTARTS   AGE
importer-centos7-clone-nfs                    1/1     Running   0          6s
virt-launcher-centos8-server-hostpath-7vbsp   1/1     Running   0          38m
virt-launcher-centos8-server-nfs-tw5kc        1/1     Running   0          93m
~~~

Be fast!

~~~bash
$ oc logs -f importer-centos7-clone-nfs
I0721 05:54:57.926925       1 importer.go:51] Starting importer
I0721 05:54:57.927727       1 importer.go:107] begin import process
I0721 05:54:58.090877       1 data-processor.go:275] Calculating available size
I0721 05:54:58.094471       1 data-processor.go:283] Checking out file system volume size.
I0721 05:54:58.094909       1 data-processor.go:287] Request image size not empty.
I0721 05:54:58.094940       1 data-processor.go:292] Target size 10Gi.
I0721 05:54:58.096037       1 util.go:37] deleting file: /data/disk.img
I0721 05:54:58.208590       1 data-processor.go:205] New phase: Convert
I0721 05:54:58.208617       1 data-processor.go:211] Validating image
I0721 05:54:58.391211       1 qemu.go:212] 0.00
I0721 05:54:58.832161       1 qemu.go:212] 1.23
I0721 05:54:59.167900       1 qemu.go:212] 2.36
I0721 05:54:59.311628       1 qemu.go:212] 3.58
...
I0721 05:55:12.209566       1 qemu.go:212] 96.81
I0721 05:55:12.669545       1 qemu.go:212] 98.00
I0721 05:55:12.719049       1 qemu.go:212] 99.22
I0721 05:55:13.098490       1 data-processor.go:205] New phase: Resize
I0721 05:55:13.112280       1 data-processor.go:268] Expanding image size to: 10Gi
I0721 05:55:13.128083       1 data-processor.go:205] New phase: Complete
I0721 05:55:13.128221       1 importer.go:160] Import complete
~~~

The bound PV:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound    default/centos7-clone-nfs                         nfs                             5h26m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Bound    default/centos8-nfs                               nfs                             5h26m
pvc-9346e93a-4c9f-4266-9b00-4b1a329aed1d   100Gi      RWX            Delete           Bound    openshift-image-registry/image-registry-storage   managed-nfs-storage             12h
pvc-9462894f-cbf5-455f-8465-709ec6d6f428   222Gi      RWO            Delete           Bound    default/centos8-hostpath                          hostpath-provisioner            4h4m
~~~

And finally our new PVC:

~~~bash
$ oc get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
centos7-clone-nfs   Bound    nfs-pv1                                    10Gi       RWO,RWX        nfs                    93s
centos8-hostpath    Bound    pvc-9462894f-cbf5-455f-8465-709ec6d6f428   222Gi      RWO            hostpath-provisioner   4h4m
centos8-nfs         Bound    nfs-pv2                                    10Gi       RWO,RWX        nfs                    5h21m
~~~

### Centos 7 Virtual Machine

Now it's time to launch our Centos 7 VM. Again we are just using the same pieces we've been using throughout the labs. For review, we are using the `centos7-clone-nfs` PVC we just prepared (created via CDI import of the Centos image, stored on NFS) -- however, as the standard, bridged networking on the workers via the `tuning-bridge-fixed` construct is not an externally routeable subnet, in this case we will be utilizing the masquerade pod network.

Additionally, we are going to run quite a bit more via cloud-init. This time we will do the following:

* set the centos user password to "redhat"
* enable ssh logins
* install podman
* set up an ngnix podman container to allow us to access a simple web page on the host

**Note**> You can find this yaml outside the lab environment at [https://github.com/heatmiser/openshift-virt-labs/tree/packet/configs/centos7-clone-nfs.yaml](https://github.com/heatmiser/openshift-virt-labs/tree/packet/configs/centos7-clone-nfs.yaml)

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
 annotations:
   kubevirt.io/latest-observed-api-version: v1alpha3
   kubevirt.io/storage-observed-api-version: v1alpha3
   name.os.template.kubevirt.io/centos7.0: CentOS 7
 name: centos7-clone-nfs
 namespace: default
 labels:
   app: centos7-clone-nfs
   flavor.template.kubevirt.io/small: 'true'
   os.template.kubevirt.io/rhel7.9: 'true'
   vm.kubevirt.io/template: rhel7-server-small-v0.11.3
   vm.kubevirt.io/template.namespace: openshift
   vm.kubevirt.io/template.revision: '1'
   vm.kubevirt.io/template.version: v0.11.3
   workload.template.kubevirt.io/server: 'true'
spec:
 running: true
 template:
   metadata:
     creationTimestamp: null
     labels:
       flavor.template.kubevirt.io/small: 'true'
       os.template.kubevirt.io/rhel7.9: 'true'
       vm.kubevirt.io/template: rhel7-server-small-v0.11.3
       vm.kubevirt.io/template.namespace: openshift
       vm.kubevirt.io/template.revision: '1'
       vm.kubevirt.io/template.version: v0.11.3
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
           - masquerade: {}
             model: virtio
             name: nic0
             ports:
               - port: 80
         rng: {}
       machine:
         type: pc-q35-rhel8.2.0
       resources:
         requests:
           memory: 2Gi
     evictionStrategy: LiveMigrate
     hostname: centos7-clone-nfs
     networks:
       - name: nic0
         pod: {}
     terminationGracePeriodSeconds: 0
     volumes:
       - name: disk0
         persistentVolumeClaim:
           claimName: centos7-clone-nfs
       - cloudInitNoCloud:
           userData: |-
             #cloud-config
             password: redhat
             chpasswd: {expire: False}
             ssh_pwauth: 1
             write_files:
               - path: /etc/systemd/system/nginx.service
                 permissions: ‘0755’
                 content: |
                     [Unit]
                     Description=Nginx Podman container
                     Wants=syslog.service
                     [Service]
                     ExecStart=/usr/bin/podman run --net=host nginxdemos/hello
                     ExecStop=/usr/bin/podman stop --all
                     [Install]
                     WantedBy=multi-user.target
             runcmd:
               - [ systemctl, restart, network ]
               - [ yum, install, -y, podman ]
               - [ systemctl, daemon-reload ]
               - [ systemctl, enable, nginx ]
               - [ systemctl, start, --no-block, nginx ]          
         name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos7-clone-nfs created
~~~

We can view the running VM and VMI:

~~~bash
$ oc get vm
oc get vm
NAME                      AGE    VOLUME
centos7-clone-nfs         52s    
centos8-server-hostpath   51m    
centos8-server-nfs        106m

$ oc get vmi
NAME                      AGE    PHASE     IP                 NODENAME
centos7-clone-nfs         63s    Running   10.254.4.15        worker-2.lab01.redhatpartnertech.net
centos8-server-hostpath   89m    Running   192.168.2.108/24   worker-2.lab01.redhatpartnertech.net
centos8-server-nfs        143m   Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

And remember how we set a few things via cloud-init? Let's see if it worked and SSH to the IP of this newly created VM. Login as the `centos` user with the password `redhat`.

$ virtctl expose vm centos7-clone-nfs --port=80 --target-port=80 --name=centos7-clone-nfs-nginx --type=NodePort 
Service centos7-clone-nfs-nginx successfully exposed for vm centos7-clone-nfs

Then, create a route to the service...

kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: nginx
  namespace: default
  selfLink: /apis/route.openshift.io/v1/namespaces/default/routes/nginx
  uid: e20ae92d-871b-4529-b522-183320b06aa6
  resourceVersion: '458918'
  creationTimestamp: '2020-09-10T10:32:25Z'
  annotations:
    openshift.io/host.generated: 'true'
  managedFields:
    - manager: Mozilla
      operation: Update
      apiVersion: route.openshift.io/v1
      time: '2020-09-10T10:32:25Z'
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:port':
            .: {}
            'f:targetPort': {}
          'f:to':
            'f:kind': {}
            'f:name': {}
            'f:weight': {}
          'f:wildcardPolicy': {}
    - manager: openshift-router
      operation: Update
      apiVersion: route.openshift.io/v1
      time: '2020-09-10T10:32:25Z'
      fieldsType: FieldsV1
      fieldsV1:
        'f:status':
          'f:ingress': {}
spec:
  host: nginx-default.apps.lab01.redhatpartnertech.net
  to:
    kind: Service
    name: centos7-clone-nfs-nginx
    weight: 100
  port:
    targetPort: 80
  wildcardPolicy: None
status:
  ingress:
    - host: nginx-default.apps.lab01.redhatpartnertech.net
      routerName: default
      conditions:
        - type: Admitted
          status: 'True'
          lastTransitionTime: '2020-09-10T10:32:25Z'
      wildcardPolicy: None
      routerCanonicalHostname: apps.lab01.redhatpartnertech.net

~~~bash
$ ssh centos@192.168.2.10
The authenticity of host '192.168.47.7 (192.168.47.7)' can't be established.
ECDSA key fingerprint is SHA256:/tdwPiOJAUQ0c43P8nD2YIj49e07CGsNe2lp/TvHqHg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.2.10' (ECDSA) to the list of known hosts.
centos@192.168.2.10's password:

[centos@centos7-clone-nfs ~]$
~~~

And check for the running nginx service:

~~~bash
[centos@centos7-clone-nfs ~]$ sudo systemctl status nginx
● nginx.service - Nginx Podman container
   Loaded: loaded (/etc/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2020-07-24 06:18:14 UTC; 26s ago
 Main PID: 8746 (podman)
   CGroup: /system.slice/nginx.service
           └─8746 /usr/bin/podman run --net=host nginxdemos/hello
(...)
~~~

Let's quickly verify that this is working as expected - you should be able to navigate directly to the IP address of your machine in your browser - recalling that in the example it's *192.168.47.7*:

> **NOTE**: You need to be using the browser you configured for the bastion's squid for this to work.

<img src="img/centos7-clone-nginx2.png"/>

> **NOTE**: These steps are important for both this lab and a future one; please ensure they complete correctly.

Remember to logout of your running VM:

~~~bash
[centos@centos7-clone-nfs ~]$ exit
logout
Connection to 192.168.47.7 closed.
$
~~~

We now are going to shutdown the VM so we can clone it without risking filesystem corruption. To do that it's time to introduce a new tool, `virtctl`.

`virtctl` is like oc for VMs. It allows for easy access to the kubervirt API allowing a for easier management of Virtual Machines. Trying running `virtctl --help` for full details. It can do a lot of clever things so check it out. For now, let's just use `virtctl` to stop the running VM so we can clone it. 

~~~bash
$ virtctl stop centos7-clone-nfs
VM centos7-clone-nfs was scheduled to stop
~~~

And the VM **instance** (VMI) is removed:

~~~bash
$ oc get vmi
NAME                      AGE    PHASE     IP                 NODENAME
centos8-server-hostpath   123m   Running   192.168.2.108/24   worker-2.lab01.redhatpartnertech.net
centos8-server-nfs        178m   Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

But of course the VM still exists:

~~~bash
$ oc get vm
NAME                      AGE    VOLUME
centos7-clone-nfs         35m    
centos8-server-hostpath   123m   
centos8-server-nfs        178m
~~~

### Clone the VM

Now that we've got a working virtual machine with a test workload we're ready to actually clone it, to prove that the built-in cloning utilities work, and that the cloned machine shares the same workload. First we need to create a PV (persistent volume) to clone into. This is done by creating a special resource called a `DataVolume`, this custom resource type is provide by CDI. DataVolumes orchestrate import, clone, and upload operations and help the process of importing data into a cluster. DataVolumes are integrated into OpenShift Virtualization.

The volume we are creating is named `centos7-clone-dv`, we'll be pulling the data from the volume ("source") `centos7-clone-nfs` and we'll tell the CDI to provision onto a specific node. We're only using this option to demonstrate the annotation, but also because we're going to clone from an NFS-based volume onto a hostpath based volume; this is important because hostpath volumes are not shared-storage based, so the location you specify here is important as that's exactly where the cloned VM will have to run:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: cdi.kubevirt.io/v1alpha1
kind: DataVolume
metadata:
  name: centos7-clone-dv
spec:
  source:
    pvc:
      namespace: default
      name: centos7-clone-nfs
  pvc:
    accessModes:
      - ReadWriteOnce
    storageClassName: hostpath-provisioner
    resources:
      requests:
        storage: 10Gi
EOF

datavolume.cdi.kubevirt.io/centos7-clone-dv created
~~~

You can watch the progress with where it will go through `CloneScheduled` and `CloneInProgress` phases along with a handy status precentage:

~~~bash
$ watch -n5 oc get datavolume
Every 5.0s: oc get datavolume

NAME               PHASE             PROGRESS   RESTARTS   AGE
centos7-clone-dv   CloneInProgress              0          14s
(...)

NAME               PHASE             PROGRESS   RESTARTS   AGE
centos7-clone-dv   CloneInProgress   7.29%      0          90s
(...)

NAME               PHASE             PROGRESS   RESTARTS   AGE
centos7-clone-dv   CloneInProgress   100.00%    0          2m22s

(Ctrl-C to stop/quit)
~~~

> **NOTE**: It may take a few minutes for the clone to start as it's got to pull the CDI clone images down too.

View all your PVCs, and the new clone:

~~~bash
$ oc get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
centos7-clone-dv    Bound    pvc-0f9fdc0a-8a95-4ec8-863c-cd9ca992bd02   29Gi       RWO            hostpath-provisioner   3m
centos7-clone-nfs   Bound    nfs-pv2                                    10Gi       RWO,RWX        nfs                    21m
centos8-hostpath    Bound    pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb   29Gi       RWO            hostpath-provisioner   170m
centos8-nfs         Bound    nfs-pv1                                    10Gi       RWO,RWX        nfs                    70m
~~~


### Start the cloned VM

Finally we can start up a new VM using the cloned PVC. A few things to note before we do this. We are going to name this VM `centos7-clone-dv` and it will use the `centos7-clone-dv` PVC we created in the previous steps which was cloned from the `centos7-clonenfs` machine. 

**Note>** You can find an external version of this yaml at [https://github.com/heatmiser/openshift-virt-labs/tree/packet/configs/centos7-clone-dv.yaml](https://github.com/heatmiser/openshift-virt-labs/tree/packet/configs/centos7-clone-dv.yaml)

First create the new VM:

~~~bash

$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
 annotations:
   kubevirt.io/latest-observed-api-version: v1alpha3
   kubevirt.io/storage-observed-api-version: v1alpha3
   name.os.template.kubevirt.io/centos7.0: CentOS 7
 name: centos7-clone-dv
 namespace: default
 labels:
   app: centos7-clone-dv
   flavor.template.kubevirt.io/small: 'true'
   os.template.kubevirt.io/rhel7.9: 'true'
   vm.kubevirt.io/template: rhel7-server-small-v0.11.3
   vm.kubevirt.io/template.namespace: openshift
   vm.kubevirt.io/template.revision: '1'
   vm.kubevirt.io/template.version: v0.11.3
   workload.template.kubevirt.io/server: 'true'
spec:
 running: true
 template:
   metadata:
     creationTimestamp: null
     labels:
       flavor.template.kubevirt.io/small: 'true'
       kubevirt.io/domain: centos7-clone-dv
       kubevirt.io/size: small
       os.template.kubevirt.io/centos7.0: 'true'
       vm.kubevirt.io/name: centos7-clone-dv
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
           - masquerade: {}
             model: virtio
             name: nic0
             macAddress: 'de:ad:be:ef:00:04'
             ports:
               - port: 80
         rng: {}
       machine:
         type: pc-q35-rhel8.2.0
       resources:
         requests:
           memory: 2Gi
     evictionStrategy: LiveMigrate
     hostname: centos7-clone-dv
     networks:
       - name: nic0
         pod: {}
     terminationGracePeriodSeconds: 0
     volumes:
       - name: disk0
         persistentVolumeClaim:
           claimName: centos7-clone-dv
       - cloudInitNoCloud:
           userData: |-
             #cloud-config
             password: redhat
             chpasswd: {expire: False}
             ssh_pwauth: 1
             write_files:
               - path: /etc/sysconfig/network-scripts/ifcfg-eth0
                 permissions: ‘0644’
                 content: |
                     BOOTPROTO=dhcp
                     DEVICE=eth0
                     HWADDR=de:ad:be:ef:00:04
                     ONBOOT=yes
                     TYPE=Ethernet
                     USERCTL=no
             runcmd:
               - [ systemctl, restart, network ]
         name: cloudinitdisk
EOF

virtualmachine.kubevirt.io/centos7-clone-dv created
~~~

After about 6 minutes you should see the new virtual machine running:

~~~bash
$ oc get vmi
NAME                      AGE     PHASE     IP                 NODENAME
centos7-clone-dv          3m32s   Running   10.254.4.25        worker-2.lab01.redhatpartnertech.net
centos8-server-hostpath   3h5m    Running   192.168.2.108/24   worker-2.lab01.redhatpartnertech.net
centos8-server-nfs        4h      Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

Now, a quick note about the use of cloud-init for this VM and how it relates to the cloning. We're using cloud-init to change the NIC file on the clone. In this case we set the MAC address with OpenShift Virtualization using the interfaces block and setting `             macAddress: 'de:ad:be:ef:00:04'`. 

We then need to use cloud-init to create the NIC config file to match. If we did not, then OpenShift Virtualization would have assumed DHCP and used our new MAC as we saw in previous examples. So while even though it's a clone, the network settings would be cleared so the machine can be managed.

But it IS a clone! And if you'll remember we installed podman and set up an NGINX server running on the source of this clone, so we'd expect to see those (and we did NOT set them with cloud-init).

Wanna, see? Let use `virtctl` to login and check ...

Login to the newly running instance (centos/redhat):

~~~bash
$ virtctl console centos7-clone-dv
Successfully connected to centos7-clone-dv console. The escape sequence is ^]

CentOS Linux 7 (Core)
Kernel 3.10.0-1127.el7.x86_64 on an x86_64

centos7-clone-nfs login: centos
Password:
Last login: Thu Sep 10 11:39:09 on tty1
[centos@centos7-clone-nfs ~]$
~~~
And look for some of the features from the source:

~~~bash
[centos@centos7-clone-dv ~]$ sudo rpm -qa | grep podman
podman-1.6.4-18.el7_8.x86_64

[centos@centos7-clone-nfs ~]$ sudo systemctl status nginx
● nginx.service - Nginx Podman container
   Loaded: loaded (/etc/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2020-09-10 11:38:44 UTC; 5min ago
 Main PID: 559 (podman)
   CGroup: /system.slice/nginx.service
           └─559 /usr/bin/podman run --net=host nginxdemos/hello
(...)
~~~

There the are! We have a Dolly (obscure sheep reference from the 90s and test to see if you are still paying attention - if you are send a sheep emoticon to the chat room for the course).

OK, go ahead and exit the host ...

~~~bash
[centos@centos7-clone-nfs ~]$ exit
logout

(issue a ctrl-] to break the session)
~~~

and check all is well with `oc`

~~~bash
$ oc get vmi
NAME                      AGE    PHASE     IP                 NODENAME
centos7-clone-dv          7m1s   Running   10.254.4.25        worker-2.lab01.redhatpartnertech.net
centos8-server-hostpath   3h8m   Running   192.168.2.108/24   worker-2.lab01.redhatpartnertech.net
centos8-server-nfs        4h3m   Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

### Test the clone some more!

Connect your browser to IP on the DV clone (in this case that is: http://192.168.47.8 and you should find the **ngnix** server you configured on the original host, prior to the clone, pleasantly serving your request:

<img src="img/dv-clone-nginx.png"/>

That's it! You've proven that your clone has worked, and that the hostpath based volume is an identical copy of the original NFS-based one.

### Clean up

Before moving on to the next lab let's clean up some VMs so we ensure our environment has all the resources it might need; we're going to delete our Centos 8 hostpath-based VM and both our Centos 7 VMs:

~~~bash
$ oc delete vm/centos7-clone-dv vm/centos8-server-hostpath vm/centos7-clone-nfs
virtualmachine.kubevirt.io "centos7-clone-dv" deleted
virtualmachine.kubevirt.io "centos8-server-hostpath" deleted
virtualmachine.kubevirt.io "centos7-clone-nfs" deleted
~~~

> **NOTE**: If you check `oc get vmi` too quickly (example below) you might see the VMs in a `Failed` state. This is normal and when you check again they should disappear accordingly.

~~~bash
$ oc get vmi
NAME                      AGE     PHASE     IP                 NODENAME
centos7-clone-dv          6m46s   Failed    192.168.47.8/24   cluster-august-lhrd5-worker-6w624
centos8-server-hostpath   66m     Failed    192.168.47.6/24   cluster-august-lhrd5-worker-6w624
centos8-server-nfs        73m     Running   192.168.47.5/24   cluster-august-lhrd5-worker-6w624

$ oc get vmi
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   73m   Running   192.168.47.5/24   cluster-august-lhrd5-worker-6w624
~~~

