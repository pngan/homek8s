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
    alias sudo='sudo '        # Allow alias to work with sudo. Note trailing space.

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
Initially the new pod is will in a `ContainerCreating` state, and then after a few seconds it will transition to the `Running` state. 

A pod is an emphemeral object (comes and goes) and the normal way to interact with the pod is use a kubernetes abstraction of the pods called a service. You can get the service IP address using the command:

```bash
k get services

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP   10.152.183.1    <none>        443/TCP          49m
ping-server   NodePort    10.152.183.23   <none>        8000:31000/TCP   39m
```

Note the ``CLUSTER-IP`` address and port of the ping-server, and access it's endpoint to get the current UTC:

```bash
curl 10.152.183.23:8000

2021-09-25 21:13:15.970323
```

## Accessing the Ping Service from another machine in the same network

In the above section the `CLUSTER-IP` (10.152.X.X) are Virtup IP's routable from within the cluster only. If you want to access the service from outside the cluster, like from another machine, you can access it another way. There are [various other ways](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) (loadbalancer, external name), but we will use a `nodeport`.

A nodeport is set up in `ping.yaml` and to try this out, go to another computer that can access the cluster machine and from a web browser within the same network as the k8s server and navigate to the URL:

```
http://<IP adresss or name of K8s machine>:31000
```

## Accessing the Ping Service via https

Web browsers require `https` to access end-points over the internet. 

This requires `cert-mananager` to retrieve certificates from `Let's Encrypt`, a certifcate issuer, and an ingress controller.

- Install cert-manager

    ```
    k apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
    ```

- Install nginx Ingress Controller

    ```bash
    k apply -f nginx-ingress-deploy.yaml

    clusterissuer.cert-manager.io/letsencrypt-prod created
    ```

- Set the NAT routing of the external modem to route `HTTP` traffic to port `30000` of the cluster machine, and `HTTPS` traffic to port `30001` of the cluster machine.  These port values are defined in `nginx-ingress-deploy.yaml`


- Apply Certificate ClusterIssuer:

    ```bash
    k apply -f clusterissuer.yaml

    <many lines not shown>
    ```

- Apply Ingress:

    ```bash
    k apply -f ingress.yaml

    ingress.networking.k8s.io/ingress-homeserver created
    ```

- Visit your site with a web browser e.g. `https://nganfamily.com` At first the certifate will show issuing, but in a minute, `HTTP01` challenge would have completed, an a valid Let's Encrypt certificate would have been issued and stored in the secret store of the k8s cluster.

<img src="https://user-images.githubusercontent.com/4557674/134793010-61edefdc-5192-4f06-a5ab-6729993ecf69.png" alt="drawing" width="300"/>