## Welcome!

Welcome to the OpenShift Virtualization self-paced lab on Packet Cloud. We've put this together to give you an overview and technical deep dive into how OpenShift Virtualization works, and how it will bring virtualization capabilities to OpenShift over the next few months.

**OpenShift Virtualization** is now the official *product name* for the Container-native Virtualization operator for OpenShift. This has more commonly been referred to as "**CNV**" and is the downstream offering for the upstream [Kubevirt project](https://kubevirt.io/). While some aspects of this lab will still have references to "CNV" any reference to "CNV", "Container-native Virtualization" and "OpenShift Virtualization" can be used interchangeably.

In these labs you'll utilize a real, baremetal UPI OpenShift 4.5 deployment on Packet's baremetal cloud. This will help get you up to speed with the concepts of OpenShift Virtualization.

This is the self-hosted lab guide that will run you through the following:

* **Validating the OpenShift deployment**
* **Deploying OpenShift Virtualization**
* **Setting up Storage for OpenShift Virtualization**
* **Setting up Networking for OpenShift Virtualization**
* **Deploying some Test Workloads**
* **Cloning Workloads**
* **Performing Live Migrations and Node Maintenance**
* **Utilizing network masquerading on pod networking for VM's**

## Lab Setup

The entire lab _can__ be run from within the hosted lab environment you are in now. The lab environment provides both a CLI with the tools you need (oc, virtctl) as well as access to the OpenShift console as a privileged user. Switching between the console and the CLI enviornment is easy. At the top middle of the lab guide you'll find a link to switch between the "**Terminal**" and the "**Console**". At times, it may be convenient to also access the OpenShift cluster via the lb/bastion system. Note, if you do use the lb/bastion node, you will need to export the localtion of the kubeconfig authorization file:

~~~bash
$ export KUBECONFIG=~/ocp4upi/artifacts/install/auth/kubeconfig
~~~

<img src="img/console-button.png"/>

Selecting either will highlight your choice in blue and change the window's focus to the requested environment. 

Within the lab you can cut and paste commands directly from the instructions; but also be sure to review all commands carefully both for functionality and syntax!

> **NOTE**: In some browsers and operating systems you may need to use Ctrl-Shift-C / Ctrl-Shift-V to copy/paste!

## Environment

The lab environment consists of three (3) OpenShift masters and three (3) OpenShift workers. 
The installation consists of two networks, one for internal OpenShift communication, and another to represent a "public" network unrelated to OpenShift. This second network is connected to all OpenShift workers. We use this public network in the labs but it should be noted that the network uses an unroutable range and is just an example. The point is that it is external to OpenShift and could be any extra network.

In additon to the OpenShift deployment we are running a bastion host with a few extra services:

* A DHCPD server to provide IP addresses to VMs configured to use the internal network.
* An NFS server to provide basic storage to our OpenShift environment. It is full configured and available to the lab.
* A Haproxy server, which load balances API end point, and application router endpoints.

Conceptually, the environment looks like this:

<center>
    <img src="img/labarch.png"/>
</center>

## Accessing the lb/bastion host

As mentioned, this host serves a few purposes and **it is required to connect to it before starting the lab**.
 

> **NOTE**: All connections to the lb/bastion are made as "`root`" with either the password supplied in the Packet admin console or via public/private key.

### Step 1 
Find the bastion's private IP

~~~bash
$ ssh lab-user@bastion.august.students.osp.opentlc.com "ip a s eth0 |grep -Po 'inet \K[\d.]+'"
lab-user@bastion.august.students.osp.opentlc.com's password:
192.168.47.16
~~~

~~~bash
$ ssh lab-user@bastion.august.students.osp.opentlc.com -L 8080:192.168.47.16:3128
lab-user@bastion.august.students.osp.opentlc.com's password:
Activate the web console with: systemctl enable --now cockpit.socket

This system is not registered to Red Hat Insights. See https://cloud.redhat.com/
To register this system, run: insights-client --register

Last login: Sun Jul 26 19:31:07 2020 from 106.69.159.19

[lab-user@bastion ~]$
~~~

> **NOTE**: You will also use the login for some non-OpenShift commands in the lab. However, the bastion has the openshift client, `oc`, configured and useable. Try `oc get nodes` when you are there!

# Feedback Please!

We very much welcome feedback on the content, what's missing, what would be good to have, etc. Please feel free to submit PR's or raise [GitHub](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/rhpds) issues! We are always excited to continue to evolve this lab to fit your requirements. So please do let us know what we can add to make it work even better for your needs.