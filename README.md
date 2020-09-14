## OpenShift Virtualization Hands-on Lab (Packet Cloud)

**Authors**: [Rhys Oxenham](mailto:roxenham@redhat.com) and [August Simonelli](mailto:asimonel@redhat.com)

**Packet Cloud revision**: [Mike Savage](mailto:savage@redhat.com)

# Welcome!

Welcome to our hands-on OpenShift Virtualization lab for Packet Cloud. 

> **NOTE**: We also have a branch that can be used on your own hardware which includes deployment scripts for a completely self-contained training. This is available from the [main branch of this lab's repo](https://github.com/RHFieldProductManagement/openshift-virt-labs/tree/master).

# Lab environment

The lab includes a self-hosted OpenShift Virtualization environment and a hands-on, self-paced lab guide based on [OpenShift homeroom](https://github.com/openshift-homeroom).

The lab content is presented in three easy to use panes consisting of the following sections: navigation, lab steps, working environment. Adjust sizing to suit!

You'll have access to an OpenShift CLI environment as well as the console.

Once deployed, all labs steps are expected to be run from *within* the workbook/lab environment; you do not need to use the lb/bastion's CLI or login to the OpenShift Console directly, however, utilizing the load-balancer/bastion node can be useful for instances where multiple CLI sessions are desirable.


# Lab content

The lab uses official Red Hat downstream components, where **OpenShift Virtualization** is now the official feature name of the packaged up [Kubevirt project](https://kubevirt.io/) within the OpenShift product. 

The lab runs you through the following OpenShift Virtualization tasks:

* **[Validating the OpenShift deployment](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/validation.md)**
* **[Deploying OpenShift Virtualization](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/deploy-cnv.md)**
* **[Setting up Storage for OpenShift Virtualization](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/storage-setup.md)**
* **[Setting up Networking for OpenShift Virtualization](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/network-setup.md)**
* **[Deploying Test Workloads](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/deploy-workloads.md)**
* **[Cloning Workloads](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/cloning.md)**
* **[Performing Live Migrations and Node Maintenance](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/live-migration.md)**
* **[Utilizing pod networking for VM's](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/masquerade.md)**
* **[Using the OpenShift Web Console with OpenShift Virtualization](https://github.com/heatmiser/openshift-virt-labs/blob/packet/docs/workshop/content/console.md)** 

As mentioned above, the entire environment is deployed within OpenShift infrastructure hosted by Packet Cloud. This means you can easily deploy the lab, follow some simple setup instructions, and you will have your own bare-metal OpenShift cluster to work on, with full admin access. 

The deployment is visualized as follows:

<center>
    <img src="docs/workshop/content/img/labarch.png"/>
</center>

Within this environment you can access all aspects of the lab through the deployed lab guide. You receive details regarding how to access the guide upon completion of the installation/deployment steps.

> **NOTE**: For the purposes of this repo and the labs themselves, any reference to "CNV", "Container-native Virtualization" and "OpenShift Virtualization", and "KubeVirt" can be used interchangeably.
installed
### Getting Started

1) Start with **[OpenShift on Packet via Terraform](https://github.com/heatmiser/openshift-packet-deploy)**

   Jump to the instructions [here](https://github.com/heatmiser/openshift-packet-deploy/blob/master/terraform/README.md)

   Once completed, you will have a full OpenShift cluster running on Packet's bare metal cloud.

2) If not already in place, [install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) on the system that was utilized to deploy OpenShift on Packet Cloud via Terraform.

3) Next, clone the heatmiser/agnosticd repo from GitHub and checkout the `packet-cnv` branch:

     ```bash
     git clone https://github.com/heatmiser/agnosticd.git
     cd agnosticd
     git checkout packet-cnv
     ```

   In the base of the agnosticd directory, you should see a script named `agnosticd_deploy_workload`.
   Edit this file, setting the variables `BASTION`, `SUBDOMAIN_BASE` and `ANSIBLE_USER_KEY_FILE` from values derived from the OpenShift on Packet deployment in step 1. Execute the script, which will run an ansible-playbook that deploys the OpenShift Virtualization Lab Workbook lab on the OpenShift cluster on Packet Cloud.

     ```bash
     chmod +x agnosticd_deploy_workload
     ./agnosticd_deploy_workload
     ```

   After several minutes, the lab deployment will be complete.  Review the output from the ansible-playbook, as it will have information regarding the deployment of the lab.

4) Open the URL for the OpenShift Virtualization Lab Workbook via web browser (the "CNV Lab Workbook" value, which looks like https://cnv-workbook.apps.your.unique.domain/)

>**NOTE**: You will need to accept the SSL warnings but you do not need to login to the workbook.

<center>
    <img src="docs/workshop/content/img/lab-cli-view.png"/>
</center>

5) Enjoy the lab!!!

### Contributing

**We very much welcome contributions and pull requests!**
