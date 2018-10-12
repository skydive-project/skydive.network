---
title: Blog
layout: blog-index
---

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