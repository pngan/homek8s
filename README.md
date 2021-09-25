# homek8s
Installation, setup, and build home server apps running on microk8s and a ubuntu host.

The purpose of this repos is to document the process of setting up a Kubernetes cluster, running  and monitoring it, and building and deploying apps to run on it. The repos is a record of learning focussing on how the system is built, rather than why it was done that way. 

The information described in this repository relates a single node `microk8s` cluster hosted on Ubuntu running on a hardware host. 

## Prerequisites

This home server setup requires a machine running the Ubuntu 20.04 OS or later, and access to the superuser account to issue `sudo` commands. 

## Install microk8s Kubernetes

Install microk8s following these [instructions](https://ubuntu.com/tutorials/install-a-local-kubernetes-with-microk8s#1-overview).

Microk8s requires the use of `microk8s kubectl`. For convenience this command can be aliased by adding the line in the `~/.bashrc` file:

    alias k='microk8s kubectl'
    alias m='microk8s'
    alias sudo='sudo '        # Allow alias to work with sudo

# Setup K9s

## First create the kube config
You need to install the kubernetes config file using the command:
```bash
cd 
mkdir .kube
cd .kube
microk8s config > config
```

## Install k9s

`sudo snap install k9s`

## Create a k9s config file

First find the location of where the config file should go:

`k9s info`

Then create an empty file shown in the `Configuration` path.

Populate the information using the values seeded from [here](https://github.com/derailed/k9s#k9s-configuration). 

Set the values:
```yaml
  currentContext: microk8s
  currentCluster: microk8s-cluster
```


Put `export KUBECONFIG=$HOME/.kube/config` into the the `.bashrc` file.

Re-run the `.bashrc` file:
```bash
source ~./bashrc
```

To check that microk8s is running, run `k9s` and  press `0` to list pods from all namespaces. There should be two `calico` pods running in the `kub-system` namespace.

## Install and test Ping service
