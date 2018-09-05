---
title: Blog
layout: blog-index
---

# [Network topology discovery with Ansible and Skydive](/blog/ansible-library.html)
## by Sylvain Afchain, 05/09/2018

Since `Skydive` already has a `Python` client [library](/documentation/api-python) I thought it was "fun" to create an `Ansible` module leveraging it being able to add
topology entities. In this blog post I will show how to use this module and how to leverage it to provide real topology information.


# [Deploy Skydive on top of OpenStack using Tripleo](/blog/tripleo.html)
## by Sylvain Afchain, 07/08/2018

Skydive supports multiple deployment ways, from containers (Kubernetes, OpenShift) to Ansible playbook. In this blog post I will explain how to
deploy Skydive on top of OpenStack using Tripleo. Support for Skydive is already integrated in TripleO since the Queens release but this support has been reworked the new config download](https://docs.openstack.org/tripleo-docs/latest/install/advanced_deployment/ansible_config_download.html) feature.
During the last few weekds, we added a TripleO job to our CI. In this blog post, I will extract some of the scripts involved in the CI job to show how to deploy the latest version of Skydive with Tripleo.


# [Flow Matrix](/blog/flow-matrix.html)
## by Nicolas Planel, 24/07/2018

Skydive Flow Matrix is a tool on top of Skydive that helps you understand which services are connecting to each other on your platform. Thanks to the Skydive SocketInfo probe, Flow Matrix will report all opened Sockets between client and server processes across hosts...