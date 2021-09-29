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

## Install microk8s add-ons

Various microK8s add-ons are required for this server. They can all be enabled in one command:

```bash
microk8s.enable dns storage metrics-server
```

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
http://<homek8s-servername>:31000
```

## Accessing the Ping Service via https

Web browsers require `https` to access end-points over the internet. 

To implement this, you must have a Domain Name registered to you, and a DNS entry that redirects traffic to that Domain Name to the incoming gateway to your K8s cluster. The gateway should then forward traffic through its NAT to the K8s cluster. Set the NAT routing to route `HTTP` traffic to port `30000` of the cluster machine, and `HTTPS` traffic to port `30001` of the cluster machine.  These port values are defined in `nginx-ingress-deploy.yaml`. 


This implementation [[1]](#ingress-footnote) uses [`cert-manager`](https://cert-manager.io/docs/installation/) to retrieve certificates from `Let's Encrypt`, and provide a qualified response to the `HTTP01` challenge issued from Let's Encrypt to validate the site. Cert-manager automatically updates the certificates before it's expiry date, so there is nothing required to be done to manually keep the certificates up to date.

- Install cert-manager

    ```
    k apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
    ```

- Install nginx Ingress Controller

    ```bash
    k apply -f nginx-ingress-deploy.yaml

    clusterissuer.cert-manager.io/letsencrypt-prod created
    ```

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

- Visit your site with a web browser e.g. `https://nganfamily.com` At first the certificate will show `issuing`, but in a minute, `HTTP01` challenge would have completed, an a valid Let's Encrypt certificate would have been issued and stored in the secret store of the k8s cluster.

    <img src="https://user-images.githubusercontent.com/4557674/134793010-61edefdc-5192-4f06-a5ab-6729993ecf69.png" alt="drawing" width="300"/>

# Centralized Logging Repository

This server will host some custom applications, some of which will write to a log repository called [`Seq`](https://datalust.co/seq). Install `Seq` by running the command:

```bash
k apply -f seqserver.yaml

persistentvolumeclaim/logdata created
service/seqserver created
service/seqserver-nodeport created
deployment.apps/seqserver created
```

Note: the `storage` microk8s add-on is required.


The `seq` logs can be viewed by browsing on the local network to:

```
http:<homek8s-servername>:32543
```

# Update DNS A record automatically

Occassionally, the ISP for the home modem will be assigned a new public IP Address. When this happens, the A DNS record for the Domain Name needs to be updated so that traffic arrives to the `homeK8s` server correctly.

This is performed by a [small worker app](https://github.com/pngan/external-ip-monitor) that checks hourly if the public IP address changes, and if so updates the DNS A record held at the `OVH` Domain Registry. This app is specific to the `OVH` Domain Name Registry, so it is not applicable in the general case.

This app requires the `dns` microk8s add-on to be enabled in order for the app to access the `seq` logging server.

This app requires the REST application key and login credentials for the `OVH` registry. These secrets are held in the form of a secrets resource, which is holds private information and so is not checked into the git repository.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ovh-secrets
type: Opaque
stringData:
  endpoint: <redacted>
  application_key: <redacted>
  application_secret: <redacted>
  consumer_key: <redacted>
``` 

To view the secrets, look at the logs generated by the pod `secrets-test` created by `secrets-test-deployment.yaml`.

Deploy the secrets resource and then the app:

```bash
k apply -f secret.yaml

secret/ovh-secrets created
secret/sqlserver-secrets created

k apply -f ipmon-deployment.yaml

persistentvolumeclaim/ipmondata created
deployment.apps/ipmon created
```

# Install Developer Edition of SQL Server

Install the Developer Edition of SQL Server 2019 as a platform to try out SQL experiments. This exposes SQL server on the nodeport `31433`.

```bash
k apply -f sqlserver.yaml

persistentvolumeclaim/sqldata created
deployment.apps/sqlserver created
service/sqlserver created
service/sqlserver-nodeport created
```

---

<a name="ingress-footnote">[1]</a>:
The primary source of information for the installation of the Ingress Controller and Ingress Configure is the Pluralsight course by Anthony Nocentino, [Configuring and Managing Kubernetes Networking, Services, and Ingress](https://app.pluralsight.com/library/courses/configuring-managing-kubernetes-networking-services-ingress/table-of-contents). 
