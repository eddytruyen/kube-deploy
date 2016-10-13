Flocker volumes for the Portable Multi-Node Kubernetes cluster
----------------------------------------------------------------
This fork of the docker-multinode project contains a small extension that integrates Kubernetes with Flocker. 
The extension ensures that the necessary Flocker environment variables and security credentials are set in the hyperkube containers of Kubernetes.

## Prerequisites

Linux machines with **Docker 1.11.0 or higher**

A working installation of Flocker on every Kubernetes node is required. The flocker control service is typically installed on the Kubernetes master node.

## Installing Flocker

See https://docs.clusterhq.com/en/latest/kubernetes-integration/manual-install.html.

There was some ambiguity in the Section [Configuring cluster authentication](https://docs.clusterhq.com/en/latest/kubernetes-integration/configuring-authentication.html).
It is the idea that you generate node certificates for each node to which volumes should be attached. You have to generate these node certificates from the node where the flockercli package is installed (typically the control service node of Flocker), and in the directory where the cluster.key key is stored. Then scp a pair of certificate and private key to each node under directory /etc/flocker and rename these files to node.crt and node.key. Also scp the cluster.crt file to each node under /etc/flocker.   

### Integration with Flocker in OpenStack to manager Cinder volumes

Flocker depends on an underlying dataset manager for attaching volumes.
I've tested Flocker with the Cinder volume service in a private Openstack cloud.

The clue is that you specify the right configuration for this dataset manager in an `agent.yml` file in the /etc/flocker/ directory. 

The configuration syntax depends on the underlying dataset manager.
For Openstack: see [`agent.yml`](flocker/agent.yml) file.

### Installing Flocker client

Install the flockerctl command at the control-service node. https://docs.clusterhq.com/en/latest/flocker-features/flockerctl.html

To run flockerctl, you need an APIUser certificate. How to create one is specified here:https://docs.clusterhq.com/en/latest/kubernetes-integration/generate-api-certificates.html. You also need to set 3 environment variables:

```
export FLOCKER_CERTS_PATH=/etc/flocker
export FLOCKER_USER=kubernetes
export FLOCKER_CONTROL_SERVICE=172.17.13.43
```

#### Bug 

There is a bug in version 1.13 of Flocker:

See: https://github.com/ClusterHQ/flocker/issues/2841

Quick fix: update file `cinder.py` in directory `/opt/flocker/lib/python2.7/site-packages/flocker/node/agents/` with new file in code repository [`flocker/cinder.py`](flocker/cinder.py).

This bug has been resolved in version 1.14


## Integrating Flocker and Kubernetes

On every Kubernetes node where Flocker volumes will be attached, one or more environment variables and files must be created in order to mutually authenticate Kubernetes and Flocker against each other: Kubernetes as a user of the Flocker cluster and the Flocker cluster as a Flocker volume manager to Kubernetes. The environment variables can be set in the shell session where you will run the master or worker scripts of Kubernetes:

* Three optional environment variables can be specified: 
  - Optional: `FLOCKER_USER_CA_DIR` should refer to the directory where the necessary keys and certificates for authenticating kubernetes and flocker are stored. This defaults to `/etc/flocker`. If you don't want that sensitve information, stored in the guest host's /etc/flocker directory, are passed to the hyperkube container, you better use another directory.  
  - Optional: the `FLOCKER_CONTROL_SERVICE_PORT` defaults to 4523, but if the Flocker control service listens on another port you must specify this.
  - Optional: The `FLOCKER_CONTROL_SERVICE_HOST` defaults to the `${MASTER_IP}`


* In the `FLOCKER_USER_CA_DIR` directory you need to store three files
  - the `cluster.crt` file. This file is the certificate of the Flocker cluster. 
  - the `kubernetes.key` and `kubernetes.crt` files. These files are the api client key and certificate that Kubernetes uses to talk to the Flocker control service. 
 
  If you have given other names to these files, then you have to specify these names in other environment variables:
  - `FLOCKER_CONTROL_SERVICE_CA_FILE` should refer to the full path to the cluster certificate file
  - `FLOCKER_CONTROL_SERVICE_CLIENT_KEY_FILE` should refer to the full path to the api key file for the API user
  - `FLOCKER_CONTROL_SERVICE_CLIENT_CERT_FILE` should refer to the full path to the api certificate file for the API user
  

### Final Tips

You have to specify a name when creating a dataset. Otherwise the kubernetes agent will not be able to find the dataset.

To create a dataset with a name you have to specify the following command

```
flockerctl create -m name=my-volume -s 50G -n 265a0498
```

Flocker will then complain to Kubernetes that it can't find the dataset by its datasetID, but this warning is skipped by Kubernetes. Next the Pod will start its containers and the volume is linked with a subdirectory of `/flocker` directory.

Curl example for the REST API of flocker

```
curl -XGET --cacert /etc/flocker/cluster.crt --cert /etc/flocker/kubernetes.crt --key /etc/flocker/kubernetes.key https://172.17.13.43:4523/v1/configuration/datasets
curl -XDELETE --cacert /etc/flocker/cluster.crt --cert /etc/flocker/kubernetes.crt --key /etc/flocker/kubernetes.key https://172.17.13.43:4523/v1/configuration/datasets/cb681701-01f6-4be0-9e08-ab12e694f915
```

#### Possible TODO's:
* Run Flocker using Docker as well.

## Overview

This guide will set up a 2-node Kubernetes cluster, consisting of a _master_ node which hosts the API server and orchestrates work
and a _worker_ node which receives work from the master. You can repeat the process of adding worker nodes an arbitrary number of
times to create larger clusters.

Here's a diagram of what the final result will look like:
![Kubernetes Single Node on Docker](k8s-docker.png)

### Bootstrap Docker

This guide uses a pattern of running two instances of the Docker daemon:
   1) A _bootstrap_ Docker instance which is used to start `etcd` and `flanneld`, on which the Kubernetes components depend
   2) A _main_ Docker instance which is used for the Kubernetes infrastructure and user's scheduled containers

This pattern is necessary because the `flannel` daemon is responsible for setting up and managing the network that interconnects
all of the Docker containers created by Kubernetes. To achieve this, it must run outside of the _main_ Docker daemon. However,
it is still useful to use containers for deployment and management, so we create a simpler _bootstrap_ daemon to achieve this.

### Versions supported

v1.2.x and v1.3.x are supported versions for this deployment.
v1.3.0 alphas and betas might work, but be sure you know what you're doing if you're trying them out.

### Multi-arch solution

Yeah, it's true. You may run this deployment setup seamlessly on `amd64`, `arm`, `arm64` and `ppc64le` hosts.
See this tracking issue for more details: https://github.com/kubernetes/kubernetes/issues/17981

v1.3.0 ships with support for amd64, arm and arm64. ppc64le isn't supported, due to a bug in the Go runtime, `hyperkube` (only!) isn't built for the stable v1.3.0 release, and therefore this guide can't run it. But you may still run Kubernetes on ppc64le via custom deployments.

hyperkube was pushed for ppc64le at versions `v1.3.0-alpha.3` and `v1.3.0-alpha.4`, feel free to try them out, but there might be some unexpected bugs.

### Options/configuration

The scripts will output something like this when starting:

```console
+++ [0611 12:50:12] K8S_VERSION is set to: v1.2.4
+++ [0611 12:50:12] ETCD_VERSION is set to: 2.2.5
+++ [0611 12:50:12] FLANNEL_VERSION is set to: 0.5.5
+++ [0611 12:50:12] FLANNEL_IPMASQ is set to: true
+++ [0611 12:50:12] FLANNEL_NETWORK is set to: 10.1.0.0/16
+++ [0611 12:50:12] FLANNEL_BACKEND is set to: udp
+++ [0611 12:50:12] RESTART_POLICY is set to: unless-stopped
+++ [0611 12:50:12] MASTER_IP is set to: 192.168.1.50
+++ [0611 12:50:12] ARCH is set to: amd64
+++ [0611 12:50:12] NET_INTERFACE is set to: eth0
```

Each of these options are overridable by `export`ing the values before running the script.

## Setup the master node

The first step in the process is to initialize the master node.

Clone the `kube-deploy` repo, and run [master.sh](master.sh) on the master machine _with root_:

```console
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ ./master.sh
```

First, the `bootstrap` docker daemon is started, then `etcd` and `flannel` are started as containers in the bootstrap daemon.
Then, the main docker daemon is restarted, and this is an OS/distro-specific tasks, so if it doesn't work for your distro, feel free to contribute!

Lastly, it launches `kubelet` in the main docker daemon, and the `kubelet` in turn launches the control plane (apiserver, controller-manager and scheduler) as static pods.

## Adding a worker node

Once your master is up and running you can add one or more workers on different machines.

Clone the `kube-deploy` repo, and run [worker.sh](worker.sh) on the worker machine _with root_:

```console
$ git clone https://github.com/kubernetes/kube-deploy
$ cd kube-deploy/docker-multinode
$ export MASTER_IP=${SOME_IP}
$ ./worker.sh
```

First, the `bootstrap` docker daemon is started, then `flannel` is started as a container in the bootstrap daemon, in order to set up the overlay network.
Then, the main docker daemon is restarted and lastly `kubelet` is launched as a container in the main docker daemon.

## Get kubectl

```
curl -sSL https://storage.googleapis.com/kubernetes-release/release/v[KUBECTL_VERSION]/bin/linux/amd64/kubectl > /usr/local/bin/kubectl
chmod +x /usr/local/bin/kubectl
```

## Addons

kube-dns and the dashboard are deployed automatically with v1.3.0

### Deploy DNS manually for v1.2.x

Just specify the architecture, and deploy via these commands:

```console
# Possible options: amd64, arm, arm64 and ppc64le
$ export ARCH=amd64

# If the kube-system namespace isn't already created, create it
$ kubectl get ns
$ kubectl create namespace kube-system

$ sed -e "s/ARCH/${ARCH}/g;" skydns.yaml | kubectl create -f -
```

### Test if DNS works

Follow [this link](https://releases.k8s.io/release-1.2/cluster/addons/dns#how-do-i-test-if-it-is-working) to check it out.
