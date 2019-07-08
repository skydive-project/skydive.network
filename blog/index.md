---
title: Blog
layout: blog-index
---

# [Collectd as Skydive metrics provider](/blog/collectd.html)
## by Sylvain Afchain, 08/07/2019
We recently introduced a first version of a `Skydive` `Collectd` plugin. This aims to leverage some `Collectd` plugin to enhance the
`Skydive` topology. This blog post will explain how the `Skydive` architecture allowed to implement it quickly and how the metrics
are reported.

# [Introducing the Skydive Kubernetes probe](/blog/k8s-probe.html)
## by Aidan Shribman, 01/05/2019

In this article I introduce the Kubernetes Probe which constructs the
topological view of the Kubernetes resources. I begin by providing the
motivation to using the probe as opposed to just using standard
tooling (such as `kubectl`). Next I walk through several use cases
demonstrated on various Kubernetes resources.

# [ What performance can I expect](/blog/performance-1.html)
## by Sylvain Afchain, 07/02/2019

In this article I'm going to try to answer to a question that often comes up about Skydive : "Ok but what will be the impact on my system". Answering this question can be complex as Skydive address multiple use cases. So for this article I'll take a very common use case which is
monitoring the topology and the interfaces metrics. Here it won't talk about packet capture, We will talk about this aspect in a further article.

# [Add non Skydive nodes in topology](/blog/topology-rules.html)
## by Masco Kaliyamoorthy, 27/11/2018

Skydive displays the network topology by receiving the network events from the skydive agents. You ever wondered how to add or display in topology diagram, a network components which is out of the skydives’ agent network or a non network entities like TOR, data store and etc. No more worries on that, thanks to the skydive ‘Topology Rules’ API.

Since version 0.20, Skydive provides Topology Rules API, can be used to create new nodes and edges and update existing nodes’ metadata. Topology Rules API divided in two APIs, node rule API and edge rule API. Node rule API is used for create a new node and update metadata of existing node. Edge rule API is used for create edge between two nodes i.e linking two nodes.

# [Discover topology using LLDP](/blog/lldp-probe.html)
## by Sylvain Baubeau, 09/10/2018

We recently added support for [automatic topology discovery using Ansible.](/blog/ansible-library)
During last hackathon, we discussed about adding an LLDP probe in a dynamic way by creating
a LLDP probe in Skydive.

Let's see how it works.

# [Capture for interfaces that don't exist (yet)](/blog/capture-future-intf.html)
## by Sylvain Afchain, 09/10/2018

I have been reached out a couple of time for a feature request which was something like :

"I would like to capture the very first packets, like DHCP, of a VM that is about to boot. It will be great to add a mechanism to start a capture for a future interface."

So yes it will be nice, and in fact it is already possible since the beginning of Skydive as it was one of the use cases that we wanted to address. Let's see how to achieve this.

# [Introduction to Skydive workflows](/blog/introduction-to-workflows.html)
## by Sylvain Baubeau, 29/08/2018

Since version 0.19, Skydive allows you to automate Skydive actions using a new type of object called `workflows`.
Let's imagine you want to test the connectivity between 2 containers. If you had to do it manually, you would have to :
- create a capture on the interface of each container
- generate some traffic using the packet injector
- use a Gremlin query to check for flows corresponding to the generated traffic
- delete the captures
In this blog post, we will see how you can script these actions using workflows.


# [Network topology discovery with Ansible and Skydive](/blog/ansible-library.html)
## by Sylvain Afchain, 05/09/2018

Since `Skydive` already has a `Python` client [library](/documentation/api-python) I thought it was "fun" to create an `Ansible` module leveraging it to add topology entities. In this blog post I will show how to use this module and how to use this module to provide real topology information.


# [Deploy Skydive on top of OpenStack using Tripleo](/blog/tripleo.html)
## by Sylvain Afchain, 07/08/2018

Skydive supports multiple deployment ways, from containers (Kubernetes, OpenShift) to Ansible playbook. In this blog post I will explain how to
deploy Skydive on top of OpenStack using Tripleo. Support for Skydive is already integrated in TripleO since the Queens release but this support has been reworked the new [config download](https://docs.openstack.org/tripleo-docs/latest/install/advanced_deployment/ansible_config_download.html) feature.
During the last few weekds, we added a TripleO job to our CI. In this blog post, I will extract some of the scripts involved in the CI job to show how to deploy the latest version of Skydive with Tripleo.


# [Flow Matrix](/blog/flow-matrix.html)
## by Nicolas Planel, 24/07/2018

Skydive Flow Matrix is a tool on top of Skydive that helps you understand which services are connecting to each other on your platform. Thanks to the Skydive SocketInfo probe, Flow Matrix will report all opened Sockets between client and server processes across hosts...
