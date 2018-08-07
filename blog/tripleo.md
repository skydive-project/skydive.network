---
title: Deploy Skydive on top of OpenStack using Tripleo
layout: blog-post
author: Sylvain Afchain
date: 07/08/2018
---

Skydive supports multiple deployment ways, from containers (Kubernetes, OpenShift) to Ansible playbook. In this blog post I will explain how to
deploy Skydive on top of OpenStack using Tripleo. Support for Skydive is already integrated in TripleO since the Queens release but this support has been reworked the new config download](https://docs.openstack.org/tripleo-docs/latest/install/advanced_deployment/ansible_config_download.html) feature.
During the last few weekds, we added a TripleO job to our CI. In this blog post, I will extract some of the scripts involved in the CI job to show how to deploy the latest version of Skydive with Tripleo.

## Tripleo Quickstart

As we'll use [Quickstart](https://docs.openstack.org/tripleo-quickstart/latest/getting-started.html) to setup the Tripleo environment we'll need to provide a config file `skydive-minimal.yaml` to enable Skydive.

{% highlight shell %}
{% raw %}
undercloud_generate_service_certificate: true
ssl_overcloud: true
overcloud_templates_path: /usr/share/openstack-tripleo-heat-templates
step_introspect: true

containerized_overcloud: true
delete_docker_cache: true

enable_pacemaker: false
network_isolation: true
network_isolation_type: "single-nic-vlans"

telemetry_args: >-
   -e {{ overcloud_templates_path }}/environments/disable-telemetry.yaml

extra_args: >-
   -e {{ overcloud_templates_path }}/environments/services/skydive-environment.yaml
   -e {{ working_dir }}/skydive.yaml

deploy_steps_ansible_workflow: false

config_download_args: >-
  -e /usr/share/openstack-tripleo-heat-templates/environments/config-download-environment.yaml
  --config-download
  --verbose
{% endraw %}
{% endhighlight %}

We need also to provide a proper Tripleo environment to enable Skydive. The following file is the
environment file `skydive.yaml` that will be used by to deploy the overcloud.

{% highlight shell %}
parameter_defaults:
  ControllerServices:
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::Clustercheck
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::GlanceApi
    - OS::TripleO::Services::HAproxy
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Keepalived
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::Keystone
    - OS::TripleO::Services::Memcached
    - OS::TripleO::Services::MySQL
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NeutronCorePlugin
    - OS::TripleO::Services::NeutronApi
    - OS::TripleO::Services::NeutronDhcpAgent
    - OS::TripleO::Services::NeutronL3Agent
    - OS::TripleO::Services::NeutronMetadataAgent
    - OS::TripleO::Services::NeutronOvsAgent
    - OS::TripleO::Services::NovaApi
    - OS::TripleO::Services::NovaConductor
    - OS::TripleO::Services::NovaMetadata
    - OS::TripleO::Services::NovaPlacement
    - OS::TripleO::Services::NovaScheduler
    - OS::TripleO::Services::Ntp
    - OS::TripleO::Services::OsloMessagingNotify
    - OS::TripleO::Services::OsloMessagingRpc
    - OS::TripleO::Services::Pacemaker
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::SwiftProxy
    - OS::TripleO::Services::SwiftDispersion
    - OS::TripleO::Services::SwiftRingBuilder
    - OS::TripleO::Services::SwiftStorage
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::SkydiveAnalyzer
    - OS::TripleO::Services::SkydiveAgent 
  ComputeServices:
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::ComputeNeutronCorePlugin
    - OS::TripleO::Services::ComputeNeutronOvsAgent
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NovaCompute
    - OS::TripleO::Services::NovaLibvirt
    - OS::TripleO::Services::NovaMigrationTarget
    - OS::TripleO::Services::Ntp
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::SkydiveAgent
  ControllerExtraConfig:
    nova::compute::libvirt::services::libvirt_virt_type: qemu
    nova::compute::libvirt::libvirt_virt_type: qemu
  Debug: true
  DockerPuppetDebug: True
{% endhighlight %}

As the purpose of our CI job is to test Skydive patches against Tripleo we use the following script which
is a modified version of our CI script. The goal of the script is to run, step by step, a Tripleo deployment
while ensuring that latest version of Skydive is used. To do so, we rebuild the Skydive Kolla images and
we push them in the local registry.  We do so by adding lines to override the Kolla image addresses in the environment file.

{% highlight shell %}
#!/bin/bash

set -e

QUICKSTART=${QUICKSTART:-/tmp/tripleo-quickstart}
CONFIG=${CONFIG:-$PWD/skydive-minimal.yaml}
VHOST=${VHOST:-127.0.0.2}
SKYDIVE_CONFIG=${SKYDIVE_CONFIG:-$PWD/skydive.yaml}

SKYDIVE_PATH=$PWD/skydive

# download latest binary
curl -Lo skydive https://github.com/skydive-project/skydive-binaries/raw/jenkins-builds/skydive-latest
chmod +x skydive

# download quickstart
sudo rm -rf ~/.quickstart/
sudo rm -rf /tmp/tripleo-quickstart
git clone https://github.com/openstack/tripleo-quickstart.git /tmp/tripleo-quickstart

# install the undercloud
pushd $QUICKSTART
bash quickstart.sh -R master --no-clone --tags all \
	--requirements quickstart-extras-requirements.txt \
	--config $CONFIG \
	-p quickstart.yml $VHOST

bash quickstart.sh -R master --no-clone --tags all \
	--config $CONFIG \
	-I --teardown none -p quickstart-extras-undercloud.yml $VHOST
popd

# download skydive source to get latest ansible playbooks
git clone https://github.com/skydive-project/skydive.git skydive.git

# copy Skydive resources to the undercloud
scp -F ~/.quickstart/ssh.config.ansible -r skydive.git undercloud:
scp -F ~/.quickstart/ssh.config.ansible skydive undercloud:

# make Skydive ansible available for mistral 
ssh -F ~/.quickstart/ssh.config.ansible undercloud "sudo cp -R /home/stack/skydive.git/contrib/ansible /usr/share/ansible/skydive-ansible"

# copy Tripleo skydive environment file
scp -F ~/.quickstart/ssh.config.ansible $SKYDIVE_CONFIG undercloud:skydive.yaml

# prepare the overcloud deployment
pushd $QUICKSTART
bash quickstart.sh -R master --no-clone --tags all \
	--config $CONFIG \
	-I --teardown none -p quickstart-extras-overcloud-prep.yml $VHOST

# rebuild kolla images and update the environment file
ssh -F ~/.quickstart/ssh.config.ansible undercloud <<'EOF'
REGISTRY=$(grep push_destination containers-prepare-parameter.yaml | head -n 1 | awk '{print $3}' | tr -d '"' )

sudo iptables -I INPUT -p tcp --dport 18888 -j ACCEPT
python -m SimpleHTTPServer 18888 &
HTTP_SERVER=$!

ADDRESS=$(ifconfig docker0 | awk '/inet /{print $2}')

rm -rf kolla
git clone https://github.com/openstack/kolla

pushd kolla
sed -i "s|https://github.com/skydive-project/skydive/releases/download/\(.*\)/skydive|http://$ADDRESS:18888/skydive|" docker/skydive/skydive-base/Dockerfile.j2
tools/build.py --registry $REGISTRY --push -b centos skydive-agent --tag devel --network_mode host --nocache
tools/build.py --registry $REGISTRY --push -b centos skydive-analyzer --tag devel --network_mode host --nocache
popd

echo "Kolla docker images pushed"

echo "  SkydiveAnsiblePlaybook: /usr/share/ansible/skydive-ansible/playbook.yml.sample" >> skydive.yaml

echo "  DockerSkydiveAnalyzerImage: $REGISTRY/kolla/centos-binary-skydive-agent:devel" >> skydive.yaml
echo "  DockerSkydiveAgentImage: $REGISTRY/kolla/centos-binary-skydive-agent:devel" >> skydive.yaml

kill $HTTP_SERVER
EOF

# start the overcloud deployment
bash quickstart.sh -R master --no-clone --tags all \
	--config $CONFIG \
	-I --teardown none -p quickstart-extras-overcloud.yml $VHOST
popd
{% endhighlight %}

# First Skydive request

Once deployed we can go to the undercloud machine.

{% highlight shell %}
ssh -F ~/.quickstart/ssh.config.ansible undercloud
{% endhighlight %}

There we need to get the controller IP where the Skydive analyzer runs.

{% highlight shell %}
source stackrc
CTLIP=$( openstack server show overcloud-controller-0 -f json | jq -r .addresses | cut -d '=' -f 2 )
{% endhighlight %}

Finally let's do our first Skydive request. The following one will return all the OpenvSwitch bridges.

{% highlight shell %}
export SKYDIVE_ANALYZER=$CTLIP:8082
./skydive client query "G.V().Has('Type', 'ovsbridge')" --username skydive --password secret
{% endhighlight %}


## Conclusion

As we just saw, most of the job is there to build and push Kolla images using the latest version of Skydive. 
Once the remaining patches are merged

- [https://review.openstack.org/#/c/586561/](https://review.openstack.org/#/c/586561/)
- [https://review.openstack.org/#/c/586575/](https://review.openstack.org/#/c/586561/)

and the 0.19 version of Skydive is released, providing the Skydive environment file 
(environments/services/skydive-environment.yaml) and adding

{% highlight shell %}
    - OS::TripleO::Services::SkydiveAnalyzer
    - OS::TripleO::Services::SkydiveAgent
{% endhighlight %}

to the `ControllerServices` section and only the agent to the `ComputeServices` section will be enough
the have Skydive deployed.

