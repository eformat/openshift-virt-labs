Live Migration is the process of moving an instance from one node in a cluster to another without interruption. The process can be manual or automatic and depends if the `evictionStrategy` strategy is set to `LiveMigrate` and the underlying node is placed into maintenance. It also depends on the storage access mode. Virtual machines must have a PersistentVolumeClaim (PVC) with a shared ReadWriteMany (RWX) access mode to be live migrated.

Live migration is an administrative function in OpenShift Virtualization. While the action is visible to all users, only admins can initiate a migration. Migration limits and timeouts are managed via the `kubevirt-config` `configmap`. For more details about limits see the [documentation](https://docs.openshift.com/container-platform/4.4/cnv/cnv_live_migration/cnv-live-migration-limits.html#cnv-live-migration-limits).

In your lab you should now have only one VM running. You can check that, and view the underlying host it is on, by looking at the virtual machine's instance with the `oc get vmi` command.

> **NOTE**: In OpenShift Virtualization, the "Virtual Machine" object can be thought of as the virtual machine "source" that virtual machine instances are created from. A "Virtual Machine Instance" is the actual running instance of the virtual machine. The instance is the object you work with that contains the IP, networking, and workloads, etc. 

~~~bash
$ oc get vmi
NAME                 AGE     PHASE     IP                 NODENAME
centos8-server-nfs   6h47m   Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

In this example we can see the `centos8-server-nfs` instance is on `worker-2.lab01.redhatpartnertech.net`. As you may recall we deployed this instance with the `LiveMigrate` `evictionStrategy` strategy on an NFS-based, RWX-enabled PVC. You can also review the instance with `oc describe` to ensure it is enabled.

~~~bash
$ oc describe vmi centos8-server-nfs | egrep -i '(eviction|migration)'
        f:evictionStrategy:
        f:migrationMethod:
  Eviction Strategy:  LiveMigrate
  Migration Method:  BlockMigration
~~~

And for the PVC

~~~bash
$ oc describe pvc/centos8-nfs | grep "Access Modes"
Access Modes:  RWO,RWX
~~~

The easiest way to initiate a migration is to create an `VirtualMachineInstanceMigration` object in the cluster directly against the `vmi` we want to migrate. 

**But Wait!**

**Once we create this object it will trigger the migration**, so first, let's just review what it looks like ***without applying it***:

~~~
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: centos8-server-nfs
~~~

It's really quite simple, we create a `VirtualMachineInstanceMigration` object and reference the `LiveMigratable ` instance we want to migrate: `centos8-server-nfs`.

> **NOTE**: If you try and re-run the same migration job as-is, it will report `unchanged`. So, name the `VirtualMachineInstanceMigration` object and include the name of the vmi as part of the overall name. Once the migration has completed successfully, delete the `VirtualMachineInstanceMigration` object.  To run a new job, re-run the same yaml as above.

Ok, let's go ahead and try this out:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job-centos8-server-nfs
spec:
  vmiName: centos8-server-nfs
EOF

virtualmachineinstancemigration.kubevirt.io/migration-job-centos8-server-nfs created
~~~

Now let's watch the migration job in action. First it will show `phase: Scheduling`

~~~bash
$ watch "oc get virtualmachineinstancemigration/migration-job-centos8-server-nfs -o yaml | tail"

Every 1.0s: oc get virtualmachineinstancemigration/migration-job-centos8-server-nfs -o yaml | tail                                                                   Thu Sep 10 20:57:47 2020

    time: "2020-09-10T06:48:34Z"
  name: migration-job
  namespace: default
  resourceVersion: "186751"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job-centos8-server-nfs
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Scheduled                           <-----------
~~~

Next `phase: TargetReady`

~~~bash

    time: "2020-09-10T06:48:38Z"
  name: migration-job
  namespace: default
  resourceVersion: "186759"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job-centos8-server-nfs
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: TargetReady                          <-----------

~~~

Then `phase: Running `

~~~bash

    time: "2020-09-10T06:48:43Z"
  name: migration-job
  namespace: default
  resourceVersion: "186801"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job-centos8-server-nfs
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Running                                 <-----------
~~~

And then to `phase: Succeeded `:

~~~bash

    time: "2020-09-10T06:48:51Z"
  name: migration-job
  namespace: default
  resourceVersion: "186873"
  selfLink: /apis/kubevirt.io/v1alpha3/namespaces/default/virtualmachineinstancemigrations/migration-job-centos8-server-nfs
  uid: d9e57fd2-6a95-4e71-aff7-f681205189e8
spec:
  vmiName: centos8-server-nfs
status:
  phase: Succeeded                           <-----------
~~~

Pretty quick!

Finally view the `vmi` object and you can see the new underlying host:

~~~bash
$ oc get vmi/centos8-server-nfs
NAME                 AGE     PHASE     IP                 NODENAME
centos8-server-nfs   6h52m   Running   192.168.2.107/24   worker-1.lab01.redhatpartnertech.net
~~~

In the above example we have moved the VM from **worker-2.lab01.redhatpartnertech.net** to **worker-1.lab01.redhatpartnertech.net** successfully. Utilizing a ~~shorter~~alternate method to verify _this_ particular migration was successful:

~~~bash
$ oc get virtualmachineinstancemigration migration-job-centos8-server-nfs -o json | jq .status.phase
"Succeeded"
~~~

...then remove the `VirtualMachineInstanceMigration` object...

~~~bash
$ oc delete virtualmachineinstancemigration/migration-job-centos8-server-nfs
virtualmachineinstancemigration.kubevirt.io "migration-job-centos8-server-nfs" deleted
~~~

...so that when a new migration job is needed for vm "centos8-server-nfs", we re-run the same yaml as was utilized for this migration.

Live Migration in OpenShift Virtualization is quite easy. If you have time, try some more migrations. Perhaps start a ping and migrate the machine back. Do you see anything in the ping to indicate the process?

When done with your tests rerun the describe command `oc describe vmi centos8-server-nfs` after running a few migrations. You'll see the object is updated with details of those actions. 

~~~bash
$ oc describe vmi centos8-server-nfs | tail -n 50
    Status:                True
    Type:                  AgentConnected
    Last Probe Time:       <nil>
    Last Transition Time:  2020-09-10T17:19:25Z
    Status:                True
    Type:                  Ready
  Guest OS Info:
    Id:              centos
    Kernel Release:  4.18.0-193.6.3.el8_2.x86_64
    Kernel Version:  #1 SMP Wed Jun 10 11:09:32 UTC 2020
    Name:            CentOS Linux
    Pretty Name:     CentOS Linux 8 (Core)
    Version:         8
    Version Id:      8
  Interfaces:
    Interface Name:  eth0
    Ip Address:      192.168.2.107/24
    Ip Addresses:
      192.168.2.107/24
      fe80::dcad:beff:feef:1/64
    Mac:             de:ad:be:ef:00:01
    Name:            tuning-bridge-fixed
  Migration Method:  BlockMigration
  Migration State:
    Completed:        true
    End Timestamp:    2020-09-10T17:19:27Z
    Migration UID:    d4200463-263b-41c8-9910-de2effe95412
    Source Node:      worker-1.lab01.redhatpartnertech.net
    Start Timestamp:  2020-09-10T17:19:25Z
    Target Direct Migration Node Ports:
      41127:                      49153
      41431:                      0
      43259:                      49152
    Target Node:                  worker-2.lab01.redhatpartnertech.net
    Target Node Address:          10.254.4.4
    Target Node Domain Detected:  true
    Target Pod:                   virt-launcher-centos8-server-nfs-scmk8
  Node Name:                      worker-2.lab01.redhatpartnertech.net
  Phase:                          Running
  Qos Class:                      Burstable
Events:
  Type    Reason           Age                   From                                                Message
  ----    ------           ----                  ----                                                -------
  Normal  PreparingTarget  61m                   virt-handler, worker-2.lab01.redhatpartnertech.net  Migration Target is listening at 10.254.4.4, on ports: 41431,43259,41127
  Normal  Created          61m (x53 over 3h48m)  virt-handler, worker-1.lab01.redhatpartnertech.net  VirtualMachineInstance defined.
  Normal  PreparingTarget  61m (x20 over 3h17m)  virt-handler, worker-2.lab01.redhatpartnertech.net  VirtualMachineInstance Migration Target Prepared.
  Normal  Migrating        61m (x6 over 3h17m)   virt-handler, worker-1.lab01.redhatpartnertech.net  VirtualMachineInstance is migrating.
  Normal  Migrated         61m (x4 over 3h17m)   virt-handler, worker-1.lab01.redhatpartnertech.net  The VirtualMachineInstance migrated to node worker-2.lab01.redhatpartnertech.net.
  Normal  Deleted          61m (x2 over 3h17m)   virt-handler, worker-1.lab01.redhatpartnertech.net  Signaled Deletion
~~~

## Node Maintenance

Building on-top of live migration, many organizations will need to perform node-maintenance, e.g. for software/hardware updates, or for decommissioning systems. During the lifecycle of a pod, it's almost a given that this will happen without compromising the workloads, but virtual machines can be somewhat more challenging given their legacy nature. Therefore, OpenShift Virtualization has a node-maintenance feature, which can force a machine to no longer be schedulable and any running workloads will automatically live migrate off of the node if the VM instance has the ability to (e.g. using shared storage) in addition to an appropriate eviction strategy.

Let's take a look at the current running virtual machines and the nodes we have available:

~~~bash
$ oc get nodes
NAME                                   STATUS   ROLES    AGE   VERSION
master-0.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
master-1.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
master-2.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
worker-0.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8
worker-1.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8
worker-2.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8

$ oc get vmi
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   10h   Running   192.168.2.107/24   worker-2.lab01.redhatpartnertech.net
~~~

In this environment, we have one virtual machine instance running on  node *worker-2.lab01.redhatpartnertech.net*. Let's mark that node for maintenance and ensure that our workload (VMI) moves to one of the other available worker nodes:

~~~bash
$ cat << EOF | oc apply -f -
apiVersion: nodemaintenance.kubevirt.io/v1beta1
kind: NodeMaintenance
metadata:
  name: worker-2-maintenance
spec:
  nodeName: worker-2.lab01.redhatpartnertech.net
  reason: "Worker Maintenance - Back Soon"
EOF

nodemaintenance.kubevirt.io/worker-2-maintenance created
~~~

> **NOTE**: You **may** lose your browser based web terminal, and you'll need to wait a few seconds for it to become accessible again (try refreshing your browser).

Now let's check the status of our environment:

~~~bash
$ oc get nodes
NAME                                   STATUS                     ROLES    AGE   VERSION
master-0.lab01.redhatpartnertech.net   Ready                      master   22h   v1.18.3+6c42de8
master-1.lab01.redhatpartnertech.net   Ready                      master   22h   v1.18.3+6c42de8
master-2.lab01.redhatpartnertech.net   Ready                      master   22h   v1.18.3+6c42de8
worker-0.lab01.redhatpartnertech.net   Ready                      worker   22h   v1.18.3+6c42de8
worker-1.lab01.redhatpartnertech.net   Ready                      worker   22h   v1.18.3+6c42de8
worker-2.lab01.redhatpartnertech.net   Ready,SchedulingDisabled   worker   22h   v1.18.3+6c42de8
~~~

And let's check where our VM went:

~~~bash
$ oc get vmi centos8-server-nfs
NAME                 AGE   PHASE     IP                 NODENAME
centos8-server-nfs   10h   Running   192.168.2.107/24   worker-1.lab01.redhatpartnertech.net
~~~

Success!

Note that the VM has been automatically live migrated to one of the other worker nodes, as per the `EvictionStrategy`. 

And similarly to how we handled `VirtualMachineInstanceMigration` objects, We can remove the maintenance flag by simply deleting the `NodeMaintenance` object:

~~~bash
$ oc get nodemaintenance
NAME                   AGE
worker-2-maintenance   18m

$ oc delete nodemaintenance/worker-2-maintenance
nodemaintenance.kubevirt.io "worker-maintenance" deleted
~~~

And the node will be `Ready` again:

~~~bash
$ oc get nodes
NAME                                   STATUS   ROLES    AGE   VERSION
master-0.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
master-1.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
master-2.lab01.redhatpartnertech.net   Ready    master   22h   v1.18.3+6c42de8
worker-0.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8
worker-1.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8
worker-2.lab01.redhatpartnertech.net   Ready    worker   22h   v1.18.3+6c42de8
~~~

Note the removal of the `SchedulingDisabled` annotation on the '`STATUS` column. 

> **NOTE**: When a node becomes active again, this does not mean that virtual machines will 'fail back' and Live Migrate back to it.