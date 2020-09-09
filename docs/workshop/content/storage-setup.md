Now that we've got the OpenShift Virtualization operator deployed, let's configure some storage for our VMs to use. 

>**NOTE**: Switch back to the "**Terminal**" view in your lab environment.

We're going to setup two different types of storage in this section, firstly standard NFS shared storage, and secondly `hostpath` storage which uses the hypervisor's local disks, this is somewhat akin to ephemeral storage provided by OpenStack.

First, make sure you're in the default project:

~~~bash
$ oc project default
Now using project "default" on server "https://172.30.0.1:443".
~~~

>**NOTE**: If you don't use the default project for the next few lab steps, it's likely that you'll run into some errors - some resources are scoped, i.e. aligned to a namespace, and others are not. Ensuring you're in the default namespace ensures that all of the coming lab steps should flow together.

Now let's setup a storage class for NFS backed volumes, utilising the `kubernetes.io/no-provisioner` as the provisioner. 

The `kubernetes.io/no-provisioner` provisioner is generally used for local storage solutions, such as NFS. However it means that **no** dynamic provisioning of volumes is done on demand of a PVC. Instead we have to manually create our PVs for any PVC's to utilise. 

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: nfs
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Delete
EOF

storageclass.storage.k8s.io/nfs created
~~~

Let's view the newly created storage class:

~~~bash
$ oc get sc
NAME                 PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nfs                  kubernetes.io/no-provisioner   Delete          Immediate              false                  13s
standard (default)   kubernetes.io/cinder           Delete          WaitForFirstConsumer   true                   19h
~~~

>**NOTE**: The second class there, called **standard** is for general, automated, PV creation. 

Next we will create some NFS-backed persistent volumes (PVs) for our VM persistent volume claims (PVCs) to utilize. However, before we do that, let's review the NFS setup we are going to use.

We have deployed a simple NFS server to the Packet loadbalancer/bastion host. This is the same machine you connect to for cluster administration. The NFS server is sharing several directories on that host. You can view the setup directly on that system (not from within the lab environment's CLI):

~~~bash
[root@lb-0 ~]# ls -l /mnt/nfs/
total 0
drwxrwxrwx 2 root root 6 Sep  9 18:13 ocp
drwxrwxrwx 2 root root 6 Sep  9 18:13 store00
drwxrwxrwx 2 root root 6 Sep  9 18:13 store01
drwxrwxrwx 2 root root 6 Sep  9 18:13 store02
drwxrwxrwx 2 root root 6 Sep  9 18:13 store03
drwxrwxrwx 2 root root 6 Sep  9 18:13 store04
drwxrwxrwx 2 root root 6 Sep  9 18:13 store05
drwxrwxrwx 2 root root 6 Sep  9 18:13 store06
drwxrwxrwx 2 root root 6 Sep  9 18:13 store07
drwxrwxrwx 2 root root 6 Sep  9 18:13 store08
drwxrwxrwx 2 root root 6 Sep  9 18:13 store09
~~~

We will use four of these directories (store01-store04), with each of these individual directories associated to a different PV, allowing us to run four VMs, each connected to specifically tailored NFS-backed storage location. We created the remaining storage directories as additional locations for you to test and experiment with.

When creating the PV's we need to tell OpenShift what IP address the NFS server is on.

**Be sure you know and use the address from your setup for the lab steps below. DO NOT CUT AND PASTE THEM WITHOUT CHANGING THE IP TO THE VALUE FOR YOUR DEPLOYMENT.**

~~~bash
[root@lb-0 ~]$ nmcli connection show bond0 | grep ipv4.addresses
ipv4.addresses:                         136.144.55.157/31, 10.67.10.129/31
~~~

As explained, the first address is a "public" or "external" (ie not controlled by OpenShift) network. In the example above, you'd use **136.144.55.157** in the PV claims below. 

Ok, with all that clear, let's create our PVs (**LAST WARNING! Remember to substitute the correct IP address and paths for each claim**) - when you create these temporary files, don't forget to edit the file with the correct IP address before applying it:

Return to the web console if you're using it, and then create nfs-pv1:

~~~bash
$ cat << EOF > nfs1.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /mnt/nfs/store01
    server: 136.144.55.157 <------ CHANGEME
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

$ vi nfs1.yaml
(update the IP with the your lb/bastion server IP address and remove the arrow/CHANGEME text)

$ oc apply -f nfs1.yaml
persistentvolume/nfs-pv1 created
~~~

Create nfs-pv2:

~~~bash
$ cat << EOF > nfs2.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv2
spec:
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    path: /mnt/nfs/store02
    server: 136.144.55.157 <------ CHANGEME
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
EOF

$ vi nfs2.yaml
(update the IP with the your lb/bastion server IP address and remove the arrow/CHANGEME text)

$ oc apply -f nfs2.yaml
persistentvolume/nfs-pv2 created
~~~

Check the PV's:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     37s
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     19s
~~~


Now let's create a new NFS-based Peristent Volume Claim (PVC). 

For this volume claim we will use a special annotation `cdi.kubevirt.io/storage.import.endpoint` which utilises the Kubernetes Containerized Data Importer (CDI). 

> **NOTE**: CDI is a utility to import, upload, and clone virtual machine images for OpenShift Virtualization. The CDI controller watches for this annotation on the PVC and if found it starts a process to import, upload, or clone. When the annotation is detected the `CDI` controller starts a pod which imports the image from that URL. Cloning and uploading follow a similar process. Read more about the Containerised Data Importer [here](https://kubevirt.io/2018/containerized-data-importer.html).

Basically we are askng OpenShift to create this PVC and use the image in the endpoint to fill it. In this case we are going to use a publically available Centos 8 image at `"https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2"`in the annotation.

In addition to triggering the CDI utility we also specify the storage class we created earlier (`nfs`) which is setting the `kubernetes.io/no-provisioner` type as described. Finally note the `requests` section. We are asking for a 10gb volume size which we ensured were available previously via nfs-pv1 and nfs-pv2.

OK, let's create the PVC with all this included!

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos8-nfs"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: nfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/centos8-nfs created
~~~

Once created, CDI triggers the importer pod automatically to take care of the conversion for you:

~~~bash
$ oc get pods
NAME                   READY   STATUS              RESTARTS   AGE
importer-centos8-nfs     0/1     ContainerCreating   0          1s
~~~

Watch the logs and you can see the process. The image is imported into the PV which is sitting on our NFS share. Watch the logs and wait a few minutes for the import to register Complete:

~~~bash
$ oc logs importer-centos8-nfs -f
I0721 01:58:35.578683       1 importer.go:51] Starting importer
I0721 01:58:35.580412       1 importer.go:107] begin import process
I0721 01:58:35.824561       1 data-processor.go:275] Calculating available size
I0721 01:58:35.827952       1 data-processor.go:283] Checking out file system volume size.
I0721 01:58:35.828387       1 data-processor.go:287] Request image size not empty.
I0721 01:58:35.828415       1 data-processor.go:292] Target size 10Gi.
I0721 01:58:35.842513       1 data-processor.go:205] New phase: Convert
I0721 01:58:35.842538       1 data-processor.go:211] Validating image
I0721 01:58:36.142987       1 qemu.go:212] 0.00
I0721 01:58:40.640927       1 qemu.go:212] 1.00
I0721 01:58:43.963355       1 qemu.go:212] 2.00
I0721 01:58:47.147158       1 qemu.go:212] 3.01
I0721 01:58:50.607386       1 qemu.go:212] 4.01
I0721 01:58:54.773651       1 qemu.go:212] 5.01
I0721 01:58:58.695689       1 qemu.go:212] 6.01
I0721 01:59:02.230260       1 qemu.go:212] 7.03
(...)
I0721 02:04:40.747989       1 qemu.go:212] 94.64
I0721 02:04:44.432514       1 qemu.go:212] 95.64
I0721 02:04:48.652009       1 qemu.go:212] 96.64
I0721 02:04:53.190578       1 qemu.go:212] 97.64
I0721 02:04:57.453249       1 qemu.go:212] 98.64
I0721 02:05:01.055319       1 qemu.go:212] 99.65
I0721 02:05:02.264574       1 data-processor.go:205] New phase: Resize
I0721 02:05:02.280021       1 data-processor.go:265] No need to resize image. Requested size: 10Gi, Image size: 10737418240.
I0721 02:05:02.280048       1 data-processor.go:205] New phase: Complete
I0721 02:05:02.280128       1 importer.go:160] Import complete
~~~

If you're quick, you can view the structure of the importer pod in another window to get some of the configuration it's using. Utilizing the lb/bastion system as a second CLI session helps for this type of usuage:

~~~bash
$ oc describe pod $(oc get pods | awk '/importer/ {print $1;}')
Name:         importer-centos8-nfs
Namespace:    default
Priority:     0
Node:         ocp-9pv98-worker-pj2dn/10.0.0.38
Start Time:   Mon, 20 Jul 2020 21:58:18 -0400
Labels:       app=containerized-data-importer
              cdi.kubevirt.io=importer
              cdi.kubevirt.io/storage.import.importPvcName=centos8-nfs
              prometheus.cdi.kubevirt.io=
(...)
    Environment:
      IMPORTER_SOURCE:       http
      IMPORTER_ENDPOINT:     https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2
      IMPORTER_CONTENTTYPE:  kubevirt
      IMPORTER_IMAGE_SIZE:   10Gi
      OWNER_UID:             31824304-0176-499c-8a28-ac663ae7a3ee
      INSECURE_TLS:          false
    Mounts:
      /data from cdi-data-vol (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bxr5x (ro)
(...)
Volumes:
  cdi-data-vol:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  centos8-nfs
    ReadOnly:   false
(...)
~~~

Here we can see the importer settings we requested through our claims, such as `IMPORTER_SOURCE`, `IMPORTER_ENDPOINT`, and`IMPORTER_IMAGE_SIZE`. 

Once this process has completed you'll notice that your PVC is ready to use:

~~~bash
$ oc get pvc
NAME          STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
centos8-nfs   Bound    nfs-pv1   10Gi       RWO,RWX        nfs            7m4s
~~~

> **NOTE**: In your environment it may be bound to `nfs-pv1` or `nfs-pv2` - we simply created two earlier for convenience, so don't worry if your output is not exactly the same here. The important part is that it's showing as `Bound`.

This same configuration should be reflected when asking OpenShift for a list of persistent volumes (`PV`), noting that one of the PV's we created earlier is still showing as `Available` and one is `Bound` to the claim we just requested:

~~~bash
$ oc get pvc
NAME          STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
centos8-nfs   Bound    nfs-pv1   10Gi       RWO,RWX        nfs            32s

$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS   REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                     109s
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                     91s
~~~

Recall that when we setup the `PV` resources we specified the location and path of the NFS server on that bastion that we wanted to utilise:

~~~basic
  nfs:
    path: /mnt/nfs/store01
    server: 136.144.55.157
~~~

If we have a look on our NFS server (ie the lb/bastion host) we can make sure that it's using this correctly

> **NOTE**: In your environment, if your image was 'pv1' rather than 'pv2', adjust the commands to match your setup (check both /mnt/nfs/store01 and /mnt/nfs/store02). 

~~~bash
[root@lb-0 ~]$ ls -l /mnt/nfs/store01/
total 387092
-rw-r--r--. 1 root root 10737418240 Jul 23 23:32 disk.img

[root@lb-0 ~]$ sudo qemu-img info /mnt/nfs/store01/disk.img
image: /mnt/nfs/store01/disk.img
file format: raw
virtual size: 10G (10737418240 bytes)
disk size: 427M

[root@lb-0 ~]$ sudo file /mnt/nfs/store01/disk.img
/mnt/nfs/store01/disk.img: DOS/MBR boot sector
~~~

We'll use this NFS-based Centos8 image when we provision a virtual machine in a future step.

## Hostpath Storage

Now let's create a second storage type based on `hostpath` storage. We'll follow a similar approach to setting this up, but we won't be using shared storage, so all of the data that we create on hostpath type volumes is essentially ephemeral, and resides on the local disk of the hypervisors (ie the OpenShift workers) themselves. As we're not using a pre-configured shared storage pool for this we need to ask OpenShift's `MachineConfigOperator` to do some work for us directly on our worker nodes.

Run the following in the terminal window - it will generate a new `MachineConfig` that the cluster will enact, recognising that we only match on the worker nodes (`machineconfiguration.openshift.io/role: worker`):

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 50-set-selinux-for-hostpath-provisioner-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 2.2.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=Set SELinux chcon for hostpath provisioner
            Before=kubelet.service

            [Service]
            Type=oneshot
            RemainAfterExit=yes
            ExecStartPre=-mkdir -p /var/hpvolumes
            ExecStart=/usr/bin/chcon -Rt container_file_t /var/hpvolumes

            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: hostpath-provisioner.service
EOF

machineconfig.machineconfiguration.openshift.io/50-set-selinux-for-hostpath-provisioner-worker created
~~~

This deploys a new `systemd` unit file on the worker nodes to create a new directory at `/var/hpvolumes` and relabels it with the correct SELinux contexts at boot-time, ensuring that OpenShift can leverage that directory for local storage. We do this via a `MachineConfig` as the CoreOS machine is immutable. You should first start to witness OpenShift starting to drain the worker nodes and disable scheduling on them so the nodes can be rebooted safely:

~~~bash
$ oc get nodes
NAME                                STATUS                     ROLES    AGE    VERSION
cluster-august-lhrd5-master-0       Ready                      master   153m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-1       Ready                      master   153m   v1.18.3+b74c5ed
cluster-august-lhrd5-master-2       Ready                      master   153m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-6w624   Ready                      worker   134m   v1.18.3+b74c5ed
cluster-august-lhrd5-worker-mh52l   Ready,SchedulingDisabled   worker   134m   v1.18.3+b74c5ed
~~~

> **NOTE**: This will take a few minutes (allow at least 15 mins) to reflect on the cluster, and causes the worker nodes to reboot. You'll witness a disruption on the lab guide functionality where you will see the consoles hang and/or display a "Closed" image. In some cases we have needed to refresh the entire browser.

<img src="img/disconnected-terminal.png" width="80%"/>

It should automatically reconnect but if it doesn't, you can try reloading the terminal by clicking the three bars in the top right hand corner:

<img src="img/reload-terminal.png" width="80%"/>

When you're able to issue commands again, make sure you're in the correct namespace again:

~~~bash
$ oc project default
Now using project "default" on server "https://172.30.0.1:443".
~~~

Before proceeding you need to **wait for the following command to return `True`** as it indicates when the `MachineConfigPool`'s worker has been updated with the latest `MachineConfig` requested (again, this will take at least 10 minutes in RHPDS): 

~~~bash
$ oc get machineconfigpool worker -o=jsonpath="{.status.conditions[?(@.type=='Updated')].status}{\"\n\"}"
True
~~~

Now we can set the HostPathProvisioner configuration itself, i.e. telling the operator what path to actually use - the systemd file we just applied merely ensures that the directory is present and has the correct SELinux labels applied to it:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: hostpathprovisioner.kubevirt.io/v1alpha1
kind: HostPathProvisioner
metadata:
  name: hostpath-provisioner
spec:
  imagePullPolicy: IfNotPresent
  pathConfig:
    path: "/var/hpvolumes"
    useNamingPrefix: "false"
EOF

hostpathprovisioner.hostpathprovisioner.kubevirt.io/hostpath-provisioner created
~~~

When you've applied this config, an additional pod will be spawned on each of the worker nodes; this pod is responsible for managing the hostpath access on the respective host; note the shorter age (42s in the example below):

~~~bash
$ oc get pods -n openshift-cnv | grep hostpath
hostpath-provisioner-dzgjz                         1/1     Running           0          12s
hostpath-provisioner-kv8qw                         1/1     Running           0          12s
hostpath-provisioner-operator-8f985f9-sln69        1/1     Running           0          6m34s
~~~

We're now ready to configure a new `StorageClass` for the HostPath based storage:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath-provisioner
provisioner: kubevirt.io/hostpath-provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF

storageclass.storage.k8s.io/hostpath-provisioner created
~~~

You'll note that this storage class **does** have a provisioner (as opposed to the previous use of `kubernetes.io/no-provisioner`), and therefore it can create persistent volumes dynamically when a claim is submitted by the user, let's validate that by creating a new hostpath based PVC and checking that it creates the associated PV:


~~~bash
$ cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: "centos8-hostpath"
  labels:
    app: containerized-data-importer
  annotations:
    cdi.kubevirt.io/storage.import.endpoint: "https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2"
spec:
  volumeMode: Filesystem
  storageClassName: hostpath-provisioner
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

persistentvolumeclaim/centos8-hostpath created
~~~

We use CDI to ensure the volume we're requesting uses the same Centos8 image we used before:

~~~bash
$ oc get pods
NAME                        READY   STATUS    RESTARTS   AGE
importer-centos8-hostpath   1/1     Running   0          6s
~~~

You can watch the output of this importer pod with 
`oc logs -f importer-centos8-hostpath`. 

~~~bash
$ oc logs -f importer-centos8-hostpath
I0724 03:43:59.574566       1 importer.go:51] Starting importer
I0724 03:43:59.574670       1 importer.go:107] begin import process
I0724 03:43:59.740542       1 data-processor.go:275] Calculating available size
I0724 03:43:59.747340       1 data-processor.go:283] Checking out file system volume size.
I0724 03:43:59.747370       1 data-processor.go:287] Request image size not empty.
I0724 03:43:59.747408       1 data-processor.go:292] Target size 10Gi.
I0724 03:43:59.762315       1 data-processor.go:205] New phase: Convert
I0724 03:43:59.762366       1 data-processor.go:211] Validating image
I0724 03:44:00.130339       1 qemu.go:212] 0.00
I0724 03:44:03.498343       1 qemu.go:212] 1.00
I0724 03:44:05.404291       1 qemu.go:212] 2.00
I0724 03:44:06.937406       1 qemu.go:212] 3.01
(...)
I0724 03:46:28.823264       1 qemu.go:212] 97.64
I0724 03:46:29.608848       1 qemu.go:212] 98.64
I0724 03:46:30.405216       1 qemu.go:212] 99.65
I0724 03:46:30.745609       1 data-processor.go:205] New phase: Resize
I0724 03:46:30.755972       1 data-processor.go:265] No need to resize image. Requested size: 10Gi, Image size: 10737418240.
I0724 03:46:30.755994       1 data-processor.go:205] New phase: Complete
I0724 03:46:30.756095       1 importer.go:160] Import complete
~~~

Allow the import to finish and check the status of the PV's:

~~~bash
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                             STORAGECLASS           REASON   AGE
nfs-pv1                                    10Gi       RWO,RWX        Delete           Bound       default/centos8-nfs                               nfs                             16m
nfs-pv2                                    10Gi       RWO,RWX        Delete           Available                                                     nfs                             16m
pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb   29Gi       RWO            Delete           Bound       default/centos8-hostpath                          hostpath-provisioner            2m45s
~~~

> **NOTE**: The capacity displayed above lists the available space on the host, not the actual size of the persistent volume when being used.

Let's look more closely at the PV's. Describe the new hostpath PV (noting that you'll need to adapt for the `uuid` in your environment):

~~~bash
$ oc describe pv/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb
Name:              pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb
Labels:            <none>
Annotations:       hostPathProvisionerIdentity: kubevirt.io/hostpath-provisioner
                   kubevirt.io/provisionOnNode: cluster-august-lhrd5-worker-6w624
                   pv.kubernetes.io/provisioned-by: kubevirt.io/hostpath-provisioner
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      hostpath-provisioner
Status:            Bound
Claim:             default/centos8-hostpath
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          29Gi
Node Affinity:
  Required Terms:
    Term 0:        kubernetes.io/hostname in [cluster-august-lhrd5-worker-6w624]
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb
    HostPathType:
Events:            <none>
~~~

There's a few important details here worth noting, namely the `kubevirt.io/provisionOnNode` annotation, and the path of the volume on that node. In the example above you can see that the volume was provisioned on *cluster-august-lhrd5-worker-6w624*, one our two worker nodes. 

Let's look more closely to verify that this truly has been created for us on the designated worker.

> **NOTE**: Adjust the following values to your environment. 

Create a debug pod and session on the node listed above:

~~~bash
$ oc debug node/cluster-august-lhrd5-worker-6w624
Starting pod/cluster-august-lhrd5-worker-6w624-debug ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.0.21
If you don't see a command prompt, try pressing enter.
sh-4.2# chroot /host
sh-4.4#
~~~

And review the PVC created in the /var/hpvolumes directory:

~~~bash
sh-4.4# ls -l /var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb/disk.img
-rw-r--r--. 1 root root 10737418240 Jul 24 03:46 /var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb/disk.img

sh-4.4# file /var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb/disk.img
/var/hpvolumes/pvc-e2f75a46-7402-4bc6-ac30-acce7acd9feb/disk.img: DOS/MBR boot sector
~~~

And don't forget to exit and terminate the debug pod:

~~~bash
sh4.4# exit
sh4.2# exit
Removing debug pod ...

~~~

Make sure that you've executed the two `exit` commands above - we need to make sure that you're back to the right shell before continuing, and aren't still inside of the debug pod.
