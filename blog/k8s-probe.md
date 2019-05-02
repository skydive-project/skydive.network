---
title: Introducing the Kubernetes Probe
layout: blog-post
author: Aidan Shribman
date: 01/05/2019
---

In this blog post, I present the Skydive v0.22.0 Kubernetes topology
probe which is now production-ready, supporting landscapes that
contain up to 1,000 pods. I cover the motivation, capabilities and
potential use-cases.

## Motivation

Kubernetes configuration is declarative, consisting of many resources
such as the network policy configuration below (a default deny all
rule):

```
apiVersion: networking.Kubernetes.io/v1
kind: NetworkPolicy
metadata:
  name: deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

Tools such as `kubectl` extract the declarative configuration,
together with extra status fields. Although this my be sufficient, it
does not explain how the network policy resource interacts with other
resources, such the pods set out below:

```
apiVersion: core/v1
kind: Pod
metadata:
  name: a
spec:
  ...
---
apiVersion: core/v1
kind: Pod
metadata:
  name: b
spec:
  ...
```

Furthermore, it may not be simple to answer trivial questions, such as
whether traffic is allowed to flow from `pod:a` to `pod:b`.

In essence, traffic is denied due to `networkpolicy:deny` governing both
`pod:a` and `pod:b`, as all reside in the default namespace and as the
`networkpolicy:deny` pod-selector is empty (indicating this applies to all pods
in the namespace).

Using a graph as set out below would help undetstand that `networkpolicy:deny`
governs `pod:a` and `pod:b`:

```
pod:a <---> networkpolicy:deny <---> pod:b
```

Generally, Kubernetes may define relations between resources in many
ways such as by:
- namespace/name
- label-selectors
- pod-selectors
- namespace-selectors
- ip-blocks

The Skydive Kubernetes probe has the ability to understand and
interpret these relationships and construct graph edges enabling the
user to better interpret the Kubernetes landscape.

## Getting Started

If you don't already have a Kubernetes cluster, you can setup a
minimal single node cluster for evaluation purposes using `minikube`.
You will find installation instructions for `minikube` at
[install-minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/).

You will need `kubectl` to interact with the cluster. Instructions on
installing `kubectl` can be found as follows:
[install-kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

Now you can setup Skydive on the cluster using `kubectl apply`:

```
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/contrib/kubernetes/skydive.yaml
```
This will result in the deployment of:
- Skydive agent (one per node): collecting host/node information via node
  probes
- Skydive analyzer (one per cluster): collecting Kubernetes topology via the
  Kubernetes probe

To expose the Skydive dashboard use `kubectl port-forward`:

```
kubectl port-forward service/skydive-analyzer 8082:8082
```
You can thereafter open your browser at `http://localhost:8082` and
view the Skydive dashboard.

## Hello World

Let us explore the Kubernetes hello world sample found here, to obtain
a taste of the probe:
[hello-minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)

We create the deployment using `kubectl create`:

```
kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
```

And expose the `hello-node` via `hello-world` service using `kubectl expose`:

```
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
```

In the Skydive dashboard, expand the cluster resource and the default
namespace.

Try the following Gremlin highlight options to explore the resources
created by this sample:

- hello-node deployment: `G.V().Has("Type", "deployment", "Name", "hello-node")`
- hello-node pods: `G.V().Has("Type", "pod", "Name", Regex("hello-node.*"))`
- hello-world service: `G.V().Has("Type", "service", "Name", "hello-world")`

## Topological Model

The topological view places Kubernetes resources within their correct
hierarchical location (for example, by placing a pod within its
namespace). The Kubernetes probe supports the following resources:

```
- cluster (top level)
  - namespace
    - configmap
    - cronjob
    - deployment
    - daemonset
    - ingress
    - job
    - networkpolicy
    - pod
      - container
    - persistentvolumeclaim
    - replicaset
    - secret
    - service
    - statefulset
  - node
  - persistentvolume
  - storageclass
```

Additional edges (non-ownership relationships) exist between
Kubernetes resources (e.g. k8s.networkpolicy to k8s.pod) or between
Kubernetes resources and non-Kubernetes resources (e.g. k8s.container
to docker).

## Exploring Topology

We will now go back to the network policy space to better understand
how Skydive Kubernetes helps to gain insights, via its graph layout. 

What is a Kubernetes network policy?
Let us first understand what a Kubernetes network policy is:
- a Kubernetes network policy consists of a set of allow/deny rules
- each rule limited to ingress, egress or both
- each rule applies to a subset of the pods

How is the Kubernetes network policy interpreted?
The Kubernetes network policy is interpreted as follows:
- allow if no policy rule exists
- allow if at least one allow policy rule exists

It is not always easy to understand which pods are governed by a
specific network policy, as there are many ways this can be defined,
such as where:
- pods are in a namespace specified by a namespace-selector
- pods are specified by a pod-selector
- pods are specified by ip-block

Open the WebUI and expand the cluster resource and then the default namespace.

We will use the bookinfo sample and focus our view by applying:
- Filter: `G.V().Has("K8s.Labels.app", "bookinfo")`
- Highlight: `k8s network`

Now we can create the bookinfo sample (enhanced with network policy)
by applying:

```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.0/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-deny.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-details.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-ratings.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-reviews-v1.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-reviews-v2.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/bookinfo/bookinfo-netpol-reviews-v3.yaml
```

In the following figure, we have set out a default `deny` network
policy resource and many specific `allow` network policy resources to
permit traffic only where specifically needed, such as between
`productpage` to `reviews`:

<p align="center">
  <a href="/assets/images/blog/k8s-netpol.png" data-lightbox="netpol" data-title="Skydive WebUI: Kubernetes Network Policies">
    <img src="/assets/images/blog/k8s-netpol.png" style="width:50%;"/>
  </a>
</p>

## Node Status Indicator

A status indicator provides a coloring of the node:
- Up indicated by WHITE
- Down indicated by RED

Open the WebUI and expand the cluster resource and then the default namespace, then apply:
- Filter: `k8s storage`
- Highlight: `G.V().Has("Name", Regex("task-pv-*"))`

Use `kubectl apply` to create the sample:

```
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/k8s/pv-volume.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/k8s/pv-claim.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/k8s/pv-pod.yaml
```
When running the sample, you will notice how the highlighted resources
are RED (down) during creation phase and later turn WHITE (up) when
operational. The final state is depicted in the following figure:

<p align="center">
  <a href="/assets/images/blog/k8s-storage.png" data-lightbox="storage" data-title="Skydive WebUI: Kubernetes Storage">
    <img src="/assets/images/blog/k8s-storage.png" style="width:50%;"/>
  </a>
</p>

## DevOps Automation

Skydive has a REST API enabling automation which utilizes the
Kubernetes topology model.

We use wordpress sample and focus our view by:
- Filter: `G.V().Has("K8s.Labels.app", "wordpress")`
- Highlight: `G.V().Has("Type", "secret", "Name", "mysql-pass").In()`

Let us now create the wordpress sample (password **abc123**):

```
kubectl create secret generic mysql-pass --from-literal password=abc123
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/k8s/mysql-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/skydive-project/skydive/v0.22.0/tests/k8s/wordpress-deployment.yaml
```

Updating passwords on any landscape is standard practice and is
typically governed by security regulations.

Next we demonstrate updating the password by re-creating the secret
resource (password **abc456**):

```
kubectl delete secret mysql-pass
kubectl create secret generic mysql-pass --from-literal password=abc456
```

We therefore recreated `secret:mysql-pass` with **abc456**. However,
the dependant pods were not recreated and still use the old password
**abc123**.

**Enter Skydive REST API.**

First, we need to install the Skydive binary:

```
wget -L0 https://github.com/skydive-project/skydive/releases/download/v0.22.0/skydive
chmod +x skydive
sudo mv skydive /usr/local/bin/skydive
```

By utilizing the REST API, we can delete all the outdated pods
with password **abc123** and allow the deployment resources to kick in
and create new replacement pods with password **abc456**:

```
skydive client query \
	"G.V().Has('Name', 'mysql-pass').In().Has('Type', 'pod')" | \
	jq '.[] | .Metadata.Name' | \
	xargs -n 1 kubectl delete pod
```

The following figure captures the state of the system straight after
deleting the old pods by our script (in WHITE before disappearing) and
after the deployment resources have automatically created the new pods
(in RED as they have not yet stabilized):

<p align="center">
  <a href="/assets/images/blog/k8s-secret.png" data-lightbox="secret" data-title="Skydive WebUI: Kubernetes Secret">
    <img src="/assets/images/blog/k8s-secret.png" style="width:50%;"/>
  </a>
</p>

## Conclusion

In this blog post we have learned how to use Skydive to gain insights
into our Kubernetes cluster, from topology exploration to status
indications. We have now finally showcased how DevOps automation can
be layered atop Skydive.
