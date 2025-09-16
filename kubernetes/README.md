# Kubernetes Documentation

This folder contains step-by-step guides and documentation for building and securing a Kubernetes cluster using k3s and Helm. The guides are designed for hands-on learning in a homelab or real-world environment, focusing on practical DevSecOps concepts.

## Contents

- **[part-1-preparation.md](part-1-preparation.md)**  
  Covers environment setup, including prerequisites, installing k3s on controller and worker nodes, and installing Helm.  
  - Details assumptions about virtualization (Proxmox), Fedora server setup, networking, and firewall requirements.
  - Guides you through initializing a multi-node k3s cluster and verifying node status.
  - Provides instructions for installing Helm, the Kubernetes package manager, and verifying its installation.

- **[part-2-monitoring.md](part-2-monitoring.md)**  
  Explains how to add monitoring to your k3s cluster using Prometheus and Grafana, tailored for k3s architecture.  
  - Describes k3s-specific considerations for exposing control plane metrics.
  - Provides configuration examples for `/etc/rancher/k3s/config.yaml` and Helm values for kube-prometheus-stack.
  - Walks through deploying Prometheus and Grafana with persistent storage, verifying access, and troubleshooting.
  - Includes tips for adapting monitoring to k3s’s SQLite datastore and unified control plane.

## Next Steps

Additional guides will be added to cover troubleshooting, security hardening, and advanced DevSecOps practices as the project evolves.