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
| skydive_analyzer_ip              | Defines analyzer listen IP                                     |
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
| os_user_domain_name              | OpenStack user domain name, used to create a service user      |
| os_project_domain_name           | OpenStack project domain name, used to create a service user   |
| os_endpoint_type                 | OpenStack endpoint type, used to create a service user         |
| os_identity_api_version          | OpenStack identity api version, used to create a service user  |
| skydive_os_auth_url              | OpenStack authentication URL used for authentication and Probe |
| skydive_os_service_user_role     | Set the role of the Skydive service user                       |
| skydive_os_service_username      | Skydive service user name                                      |
| skydive_os_service_password      | Skydive service password                                       |
| skydive_os_service_tenant_name   | Skydive service tenant name                                    |
| skydive_os_service_domain_name   | Skydive service domain name                                    |
| skydive_os_service_region_name   | Skydive service region name                                    |
| skydive_os_service_endpoint_type | Skydive service endpoint type                                  |
| skydive_os_service_insecure      | Set to true if allowing using insecure TLS connection          |
| skydive_auth_os_tenant_name      | Keystone tenant name that the users have to belong to          |
| skydive_auth_os_domain_name      | Keystone domain name that the users have to belong to          |
| skydive_auth_os_domain_id        | Keystone domain id that the users have to belong to            |


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

## Configuration

For a single node setup, the configuration file is optional. For a multiple
nodes setup, the analyzer IP/Port and the analyzers list need to be adapted.

<a href="/tutorials/first-steps-1.html">Tutorials</a>
describe single node deployment and multiple nodes deployment

See the full list of configuration parameters in the sample configuration file
<a href="https://github.com/skydive-project/skydive/blob/master/etc/skydive.yml.default" target="_blank">etc/skydive.yml.default</a>.

### TLS

To secure communication between Agent(s) and Analyzer, Skydive relies on TLS communication with strict cross validation.
TLS communication can be enabled by defining X509 certificates in their respective section in the configuration file, like :

{% highlight shell %}
analyzer:
  X509_cert: /etc/ssl/certs/analyzer.domain.com.crt
  X509_key:  /etc/ssl/certs/analyzer.domain.com.key

agent:
  X509_cert: /etc/ssl/certs/agent.domain.com.crt
  X509_key:  /etc/ssl/certs/agent.domain.com.key
{% endhighlight %}

#### Generate the certificates

Certificate Signing Request (CSR) :

{% highlight shell %}
openssl genrsa -out analyzer/analyzer.domain.com.key 2048
chmod 400 analyzer/analyzer.domain.com.key
openssl req -new -key analyzer/analyzer.domain.com.key -out analyzer/analyzer.domain.com.csr -subj "/CN=skydive-analyzer" -config skydive-openssl.cnf
{% endhighlight %}

Analyzer (Server certificate CRT) :

{% highlight shell %}
yes '' | openssl x509 -req -days 365  -signkey analyzer/analyzer.domain.com.key -in analyzer/analyzer.domain.com.csr -out analyzer/analyzer.domain.com.crt -extfile skydive-openssl.cnf -extensions v3_req
chmod 444 analyzer/analyzer.domain.com.crt
{% endhighlight %}

Agent (Client certificate CRT) :

{% highlight shell %}
openssl genrsa -out agent/agent.domain.com.key 2048
chmod 400 agent/agent.domain.com.key
yes '' | openssl req -new -key agent/agent.domain.com.key -out agent/agent.domain.com.csr -subj "/CN=skydive-agent" -config skydive-openssl.cnf
openssl x509 -req -days 365 -signkey agent/agent.domain.com.key -in agent/agent.domain.com.csr -out agent/agent.domain.com.crt -extfile skydive-openssl.cnf -extensions v3_req
{% endhighlight %}

skydive-openssl.cnf :

{% highlight shell %}
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req

[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = FR
stateOrProvinceName = State or Province Name (full name)
stateOrProvinceName_default = Paris
localityName = Locality Name (eg, city)
localityName_default = Paris
organizationalUnitName	= Organizational Unit Name (eg, section)
organizationalUnitName_default	= Skydive Team
commonName = skydive.domain.com
commonName_max	= 64

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:TRUE
keyUsage = digitalSignature, keyEncipherment, keyCertSign
extendedKeyUsage = serverAuth,clientAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = agent.domain.com
DNS.2 = analyzer.domain.com
DNS.3 = localhost
IP.1 = 192.168.1.1
IP.2 = 192.168.69.14
IP.3 = 127.0.0.1
{% endhighlight %}

### RBAC

Skydive's roles and policies enforcement is based on
<a href="http://github.com/casbin/casbin" target="_blank">casbin.</a> Policies
are stored in `etcd` and are therefore shared between all the analyzers. The
default model, based on `subject`s, `object`s and `action`s, can be overriden
in the `rbac` section of the configuration file: <a href="https://github.com/skydive-project/skydive/blob/master/etc/skydive.yml.default" target="_blank">here.</a>

An example policy could be:

{% highlight shell %}
p, guests, alert, read, allow
p, guests, capture, read, allow
p, guests, topology, read, allow
g, bob, guests
g, alice, guests
p, alice, capture, write, allow
p, bob, topology, read, deny
{% endhighlight %}

This policy would create a `guests` group that has the rights to read alerts and
captures, and to query the topology. `alice` and `bob` would be 2 members of this
group. In addition, `alice` would be able to capture traffic while `bob` would
not have the right to query topology.

To upload the policy, create a `mypolicy.csv` file with the above content, then
use:

{% highlight shell %}
curl -X PUT --data-urlencode "value@mypolicy.csv" http:/localhost:12379/v2/keys/casbinPolicy
{% endhighlight %}

A predefined policy is bundled into Skydive and contains all the possible
objects and actions:
see <a href="https://github.com/skydive-project/skydive/blob/master/rbac/policy.csv" target="_blank">here.</a>
