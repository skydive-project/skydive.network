---
title: Documentation
section: Deployment
layout: doc
---

## Kubernetes

Skydive provides a Kubernetes
<a href="https://github.com/skydive-project/skydive/blob/master/contrib/kubernetes/skydive.yaml" target="_blank">
  file
</a>
which can be used to deploy Skydive. It will deploy an Elasticsearch,
a Skydive analyzer and Skydive Agent on each Kubernetes nodes. Once you will
have Skydive deployment on top on your Kubernetes cluster you will be able to
monitor, capture, troubleshoot your container networking stack.

A skydive Analyzer
<a href="http://kubernetes.io/docs/user-guide/services/" target="_blank">
  Kubernetes service
</a>
is created and exposes ports for Elasticsearch and the Analyzer:

* Elasticsearch: 9200
* Analyzer: 8082

<a href="http://kubernetes.io/docs/admin/daemons/" target="_blank">
  Kubernetes DaemonSet
</a>
is used for Agents in order to have one Agent per node.

{% highlight shell %}
kubectl create -f skydive.yaml
{% endhighlight %}

## Ansible

This repository contains [Ansible](https://www.ansible.com/) roles and
playbooks to install Skydive analyzers and agents.

### Requirements

- Ansible >= 2.4.3.0
- Jinja >= 2.7
- Passlib >= 1.6

## All-in-one localhost Installation

{% highlight shell %}
sudo ansible-playbook -i inventory/hosts.localhost playbook.yml.sample
{% endhighlight %}

### Deployment mode

There are two main deployment modes available :

* binary (default)
* container

The binary mode download the latest stable version of Skydive.
The container mode uses the latest Docker container built. Docker
has to be deployed on the hosts to use this mode.

In order to select the mode, the `skydive_deployment_mode` has to be
set.

### Roles

Basically there are only two roles :

- skydive_analyzer
- skydive_agent

### Variables available

| Variable                         | Description                                                    |
| -------------------------------- | -------------------------------------------------------------- |
| skydive_topology_probes          | Defines topology probes used by the agents                     |
| skydive_fabric                   | Fabric definition                                              |
| skydive_etcd_embedded            | Use embedded Etcd (yes/no)                                     |
| skydive_etcd_port                | Defines Etcd port                                              |
| skydive_etcd_servers             | Defines Etcd servers if not embedded                           |
| skydive_analyzer_port            | Defines analyzer listen port                                   |
| skydive_analyzer_ip              | Defines anakyzer listen IP                                     |
| skydive_deployment_mode          | Specify the deployment mode                                    |
| skydive_auth_type                | Specify the authentication type                                |
| skydive_basic_auth_file          | Secret file for basic authentication                           |
| skydive_username                 | Username used for the basic authentication                     |
| skydive_password                 | Password used for the basic authentication                     |
| skydive_config_file              | Specify the configuration path                                 |
| skydive_flow_protocol            | Specify the flow protocol used                                 |
| skydive_extra_config             | Defines any extra config parameter                             |
| skydive_nic                      | Specify the listen interface                                   |
| os_auth_url                      | OpenStack authentication URL, used to create a service user    |
| os_username                      | OpenStack username, used to create a service user              |
| os_password                      | OpenStack password, used to create a service user              |
| os_tenant_name                   | OpenStack tenant name, used to create a service user           |
| os_domain_name                   | OpenStack domain name, used to create a service user           |
| os_endpoint_type                 | OpenStack endpoint type, used to create a service user         |
| skydive_os_auth_url              | OpenStack authentication URL used for authentication and Probe |
| skydive_os_service_user_role     | Set the role of the Skydive service user                       |
| skydive_os_service_username      | Skydive service user name                                      |
| skydive_os_service_password      | Skydive service password                                       |
| skydive_os_service_tenant_name   | Skydive service tenant name                                    |
| skydive_os_service_domain_name   | Skydive service domain name                                    |
| skydive_os_service_region_name   | Skydive service region name                                    |
| skydive_os_service_endpoint_type | Skydive service endpoint type                                  |
| skydive_auth_os_tenant_name      | Keystone tenant name that the users have to belong to          |
| skydive_auth_os_domain_name      | Keystone domain name that the users have to belong to          |


### How to configure Skydive

Every configuration parameter of the Skydive configuration file can be
overridden through an unique Ansible variable : `skydive_extra_config`.

To activate both `docker` and `socketinfo` probe you can use :

{% highlight shell %}
skydive_extra_config={'agent.topology.probes': ['socketinfo', 'docker'], 'logging.level': 'DEBUG'}
{% endhighlight %}

### Examples

Some examples are present in the
[inventory](https://github.com/skydive-project/skydive/tree/master/contrib/ansible/inventory) folder.
