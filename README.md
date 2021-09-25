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

# Install and Test the Ping service

Start simple, and install a ping service:

```bash
k apply -f ping.yaml

deployment.apps/ping-server created
service/ping-server created
```

Check that the ping pod is created:

```bash
k get pods

NAME                           READY   STATUS    RESTARTS   AGE
ping-server-8587f5677d-9pg66   1/1     Running   0          86s
```
Initially the new pod is will in a Creating state, and then after a few seconds it will transition to a Running state. 

A pod is an emphemeral objects (comes and goes) and the normal way to interact with the pod is via a kubernetes service. You can see list the service using the command:

```bash
k get services

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.152.183.1    <none>        443/TCP    15m
ping-server   ClusterIP   10.152.183.23   <none>        8000/TCP   5m9s
```

Note the IP address of the ping-server, and access it's endpoint to get the current time:

```bash
curl 10.152.183.23:8000

2021-09-25 21:13:15.970323
```
