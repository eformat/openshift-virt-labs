Let's first check the nodes' status; use the web-based terminal on the right:

~~~bash
$ oc get nodes
NAME                                   STATUS   ROLES    AGE    VERSION
master-0.lab01.redhatpartnertech.net   Ready    master   61m   v1.18.3+6c42de8
master-1.lab01.redhatpartnertech.net   Ready    master   61m   v1.18.3+6c42de8
master-2.lab01.redhatpartnertech.net   Ready    master   61m   v1.18.3+6c42de8
worker-0.lab01.redhatpartnertech.net   Ready    worker   42m   v1.18.3+6c42de8
worker-1.lab01.redhatpartnertech.net   Ready    worker   42m   v1.18.3+6c42de8
worker-2.lab01.redhatpartnertech.net   Ready    worker   42m   v1.18.3+6c42de8
~~~

> **NOTE**: You may see different naming pattern than above, most likely with your cluster domain name as the base of the hostnames. What is most important is you see 3 masters and 3 workers.

Next, let's validate the version that we've deployed, and the status of the cluster operators:

~~~bash
$  oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.5.8     True        False         34m    Cluster version is 4.5.8
~~~

This cluster is a 4.5.8 deployment and is currently stable (i.e. it is **not** showing as "progessing"). Let's next review the cluster operators and their status. We should expect them to all be available and also not "progressing" or "degraded."

~~~bash
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.5.8     True        False         False      35m
cloud-credential                           4.5.8     True        False         False      64m
cluster-autoscaler                         4.5.8     True        False         False      47m
config-operator                            4.5.8     True        False         False      47m
console                                    4.5.8     True        False         False      38m
csi-snapshot-controller                    4.5.8     True        False         False      43m
dns                                        4.5.8     True        False         False      61m
etcd                                       4.5.8     True        False         False      60m
image-registry                             4.5.8     True        False         False      43m
ingress                                    4.5.8     True        False         False      43m
insights                                   4.5.8     True        False         False      57m
kube-apiserver                             4.5.8     True        False         False      59m
kube-controller-manager                    4.5.8     True        False         False      59m
kube-scheduler                             4.5.8     True        False         False      59m
kube-storage-version-migrator              4.5.8     True        False         False      43m
machine-api                                4.5.8     True        False         False      54m
machine-approver                           4.5.8     True        False         False      59m
machine-config                             4.5.8     True        False         False      46m
marketplace                                4.5.8     True        False         False      42m
monitoring                                 4.5.8     True        False         False      41m
network                                    4.5.8     True        False         False      61m
node-tuning                                4.5.8     True        False         False      61m
openshift-apiserver                        4.5.8     True        False         False      46m
openshift-controller-manager               4.5.8     True        False         False      56m
openshift-samples                          4.5.8     True        False         False      46m
operator-lifecycle-manager                 4.5.8     True        False         False      61m
operator-lifecycle-manager-catalog         4.5.8     True        False         False      61m
operator-lifecycle-manager-packageserver   4.5.8     True        False         False      41m
service-ca                                 4.5.8     True        False         False      61m
storage                                    4.5.8     True        False         False      56m
~~~


### Making sure OpenShift works

OK, so this may be something that you've done before, and it's hardly very exciting, but let's have a little bit of fun. Let's deploy a nifty little application inside of a pod and use it to verify that the OpenShift cluster is functioning properly; this will involve building an application from source and exposing it to your web-browser. We'll use the **s2i** (source to image) container type:

~~~bash
$ oc project default
Now using project "default" on server "https://api.lab01.redhatpartnertech.net:6443".

$ oc new-app \
	nodeshift/centos7-s2i-nodejs:12.x~https://github.com/vrutkovs/DuckHunt-JS

(...)

--> Creating resources ...
    imagestream.image.openshift.io "centos7-s2i-nodejs" created
    imagestream.image.openshift.io "duckhunt-js" created
    buildconfig.build.openshift.io "duckhunt-js" created
    deploymentconfig.apps.openshift.io "duckhunt-js" created
    service "duckhunt-js" created
--> Success
    Build scheduled, use 'oc logs -f bc/duckhunt-js' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose svc/duckhunt-js'
    Run 'oc status' to view your app.
~~~

Our application will now build from source, you can watch it happen with:

~~~bash

$ oc logs duckhunt-js-1-build -f
(...)

Successfully pushed image-registry.openshift-image-registry.svc:5000/default/duckhunt-js:latest@sha256:4d0186040826a4be9d678459c5d6831e107a60c403d65a0da77fb076ff89084c
Push successful
~~~

> **NOTE**: You may get an error saying "Error from server (BadRequest): container "sti-build" in pod "duckhunt-js-1-build" is waiting to start: PodInitializing"; you were just too quick to ask for the log output of the pods, simply re-run the command.

You can check if the Duckhunt pod has finished building and is `Running`, if it's still showing as `ContainerCreating` just give it a few more seconds:

~~~bash
$ oc get pods
NAME                   READY   STATUS      RESTARTS   AGE
duckhunt-js-1-build    0/1     Completed   0          19m
duckhunt-js-1-deploy   0/1     Completed   0          17m
duckhunt-js-1-qcskl    1/1     Running     0          17m <-- this is the one!
~~~

Now expose the application (via the service) so we can route to it from the outside...

~~~bash
$ oc expose svc/duckhunt-js
route.route.openshift.io/duckhunt-js exposed

$ oc get route duckhunt-js
NAME          HOST/PORT                                                     PATH         SERVICES      PORT       TERMINATION   WILDCARD
duckhunt-js   duckhunt-js-default.apps.lab01.redhatpartnertech.net          duckhunt-js  8080-tcp                               None
~~~

You should be able to open up the application in the same browser that you're reading this guide from - copy and paste the link from the route output listed under "HOST/PORT". If your OpenShift cluster is working as expected and the application build was successful, you should now be able to have a quick play with this... good luck ;-)

<img src="img/duckhunt.png"/>

Now, if you can tear yourself away from the game, let's actually start working with OpenShift Virtualization. Before we do, let's clean up the default project ...

~~~bash
$ oc delete deployment/duckhunt-js bc/duckhunt-js svc/duckhunt-js route/duckhunt-js
deployment.apps "duckhunt-js" deleted
buildconfig.build.openshift.io "duckhunt-js" deleted
service "duckhunt-js" deleted
route.route.openshift.io "duckhunt-js" deleted
~~~
