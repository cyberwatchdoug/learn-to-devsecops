# Part 1: Preparation of the Environment

In Part 1 of this series, you will prepare the foundational environment required for a Kubernetes cluster using k3s and Helm. Each step will guide you through the following:

- Prerequisites and assumptions for the environment
- Installing k3s on both a Controller Node and Worker Node
- Installing Helm

## Step 0: Prerequisites and Assumptions

For this project, there are several assumptions for the environment detailed below. This are important steps, but beyond the scope of this project.

- Proxmox is already installed and fully configured on 2 separate devices
- Virtual Machines running Fedora server linux are fully configured
  - each VM has static networking IP addresses
  - each VM has unique hostnames
  - each VM has NTP setup and configured
- Both Proxmox hypervisors and their associated VM's can communicate
  - Network and firewall configurations completed
  - k3s requires some specific firewall rules be setup, such as port 6443. [more info here](https://docs.k3s.io/installation/requirements?os=rhel#operating-systems)

## Step 1: Install k3s on Controller Node

Pick a VM that you will designate as the controller node for the k8s cluster.

**What does the controller node do?**

Manages the Kubernetes cluster. In other words, it's the *brains* of the cluster. It hosts the controller plane, which is responsible for:

- Orchestration
- Scheduling
- Maintains desired states

In k3s, most of this is streamlined for learning, however the underlying responsibilities match that of enterprise k8s control planes.

To install, run the official installation script:

```bash
sudo curl -sfL https://get.k3s.io | sh -
```

After install, verify the cluster status. Yes, just one node is still considered a cluster!

```bash
sudo kubectl get nodes
```

## Step 2: Install k3s on Worker Node

On your other VM, we'll go through a similar installation process. But for this you'll need some key pieces of information in order to join the worker node with your already installed controller node:

- Controller Node's IP address
  - retrieved with `ip a`
- Controller Node's join token
  - found in `/var/lib/rancher/k3s/server/node-token`

To install, and join to the controller node cluster:

```bash
sudo curl -sfL https://get.k3s.io | K3S_URL=https://<controller-ip>:6443 K3S_TOKEN=<token> sh -
```

Confirm success by checking nodes:

```bash
sudo kubectl get nodes
```

Both nodes should now be listed.

You now have establish multi-node capability, giving you architecture to separate workloads from orchestration (if you choose). This is essential for enterprise production scaling, reliability, and separation of concerns.

## Step 3: Install Helm

Helm is the package manager for Kubernetes. It helps to simplify the deployment and management of applications on k8s clusters through packaging applications/services into reusable units called Charts. Learning Helm enables rapid, repeatable deployments as well as configuration management.

It's similar to package managers on Linux, namely `apt` or `dnf`.

You can install Helm on the same workstation you plan to run your kubectl commands. For a homelab environment, this is easiest on your controller node. However, with some configuration setup, you could also run this on other workstations.

To install Helm locally:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Optionally, you can read through the full script above before executing it. It is well documented, as pointed out by the official Helm documentation.

Confirm Helm installation:
```bash
helm version
```

## Next Steps

Congratulations! You have now laid the foundation for a robust Kubernetes environment using k3s and Helm!

With your controller and worker nodes configured, and Helm installed, your cluster is ready for deploying and managing containerized applications.

This setup not only mirros real-world enterprise practices but also prepares you for more advanced topics.

As this is a Part 1 in my series, there will be following parts to build on this environment to further develop DevOps and Security skills.