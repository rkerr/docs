---
title: Set Up Your KVM Virtual Machine for an On-premises Server Cluster
author: Cumulus Networks
weight: 74
toc: 5
bookhidden: true
---
First configure the VM on the master node, and then configure the VM on each worker node.

Follow these steps to setup and configure your VM on a cluster of servers in an on-premises deployment:

1. Verify that your master node meets the VM requirements.

    {{<netq-install/vm-reqs deployment="onprem" hypervisor="kvm">}}

2. Confirm that the needed ports are open for communications.

    {{<netq-install/port-reqs server="cluster">}}

3. Download the NetQ Platform image.

    {{<netq-install/kvm-platform-image deployment="onprem">}}

4. Setup and configure your VM.

    {{<netq-install/vm-setup hypervisor="kvm">}}

5. Verify the master node is ready for installation. Fix any errors indicated before installing the NetQ software.

    {{<netq-install/verify-cmd deployment="onprem">}}

6. Run the Bootstrap CLI on the master node. Be sure to replace the *eth0* interface used in this example with the interface on the server used to listen for NetQ Agents.

    {{<netq-install/bootstrap server="single">}}

7. Verify that your first worker node meets the VM requirements, as described in Step 1.

8. Confirm that the needed ports are open for communications, as described in Step 2.

9. Copy the platform image file you downloaded in Step 3 for the master node to your worker node.

10. Open your hypervisor and setup the VM in the same manner as for the master node.

    {{<notice note>}}
Make a note of the private IP address you assign to the worker node. It is needed for later installation steps.
    {{</notice>}}

11. Verify the worker node is ready for installation. Fix any errors indicated before installing the NetQ software.

    {{<netq-install/verify-cmd deployment="onprem">}}

12. Run the Bootstrap CLI on the worker node. Be sure to replace the *eth0* interface used in this example with the interface on the server used to listen for NetQ Agents.

    {{<netq-install/bootstrap server="cluster">}}

13. Repeat Steps 7 through 12 for each additional worker node you want in your cluster.

The final step is to install and activate the Cumulus NetQ software. You can do this using the Admin UI or the CLI.

Click the installation and activation method you want to use to complete installation:

- {{<link title="Install NetQ Using the Admin UI" text="Use the Admin UI">}} (recommended)
- {{<link title="Install NetQ Using the CLI" text="Use the CLI">}}
