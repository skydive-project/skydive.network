---
title: Documentation
section: Getting-started
layout: doc
---

There are multiple ways to easily deploy Skydive, in this section we are going
to explain the most common ways.

## Downloading binary

The easiest way is to download a static binary of Skydive. There
are two kind of binaries, one is built each time a feature or a bug fix is
available
<a href="https://github.com/skydive-project/skydive-binaries/blob/jenkins-builds/skydive-latest.gz" target="_blank">
  (continuous binary)
</a>, the others are provided for each
<a href="https://github.com/skydive-project/skydive/releases" target="_blank">
  release
</a>.

Since Skydive uses the same binary for all its component, one can use it as
agent, analyzer or client.

### All-in-One mode

This mode start an analyzer and an agent at once.

{% highlight shell %}
$ skydive allinone [--conf etc/skydive.yml]
{% endhighlight %}

### Agent and Analyzer separately

{% highlight shell %}
skydive agent [--conf etc/skydive.yml]
{% endhighlight %}

{% highlight shell %}
skydive analyzer [--conf etc/skydive.yml]
{% endhighlight %}

### Client

{% highlight shell %}
skydive client
{% endhighlight %}

### Helm

If you are using Kubernetes then you can deploy skydive using helm directly from Git:

```console
helm plugin install https://github.com/aslafy-z/helm-git --version 0.10.0
helm repo add skydive git+https://github.com/skydive-project/skydive@charts
helm repo update
helm install skydive-analyzer skydive/skydive-analyzer
helm install skydive-agent skydive/skydive-agent
kubectl port-forward service/skydive-analyzer 8082:8082
```

Open a browser to http://localhost:8082 to access the analyzer Web UI.

## Vagrant deployment

You can use Vagrant to deploy a Skydive environment with one virtual machine
running both Skydive analyzer and Elasticsearch, and two virtual machines with the
Skydive agent. This `Vagrantfile`, hosted in `contrib/vagrant` of the Git
repository, makes use of the
<a href="https://github.com/vagrant-libvirt/vagrant-libvirt" target="_blank">
libvirt Vagrant provider]</a> and uses Fedora as the box image.

{% highlight shell %}
cd contrib/vagrant
vagrant up
{% endhighlight %}

## Docker

A Docker image is available on the
<a href="https://hub.docker.com/r/skydive/" target="_blank">
  Skydive Docker Hub account
</a>.

To start the analyzer :

{% highlight shell %}
docker run -p 8082:8082 skydive/skydive analyzer
{% endhighlight %}

To start the agent :

{% highlight shell %}
docker run --privileged --pid=host --net=host -p 8081:8081 \
  -e SKYDIVE_ANALYZERS=localhost:8082 \
  -v /var/run/docker.sock:/var/run/docker.sock skydive/skydive agent
{% endhighlight %}

## Docker Compose

<a href="https://docs.docker.com/compose/" target="_blank">Docker Compose</a>
can also be used to automatically start an Elasticsearch container,
a Skydive analyzer container and a Skydive agent container. The service
definition is located in the `contrib/docker` folder of the Skydive sources.

{% highlight shell %}
docker-compose up
{% endhighlight %}

## Openstack/Devstack

Skydive provides a DevStack plugin that can be used in order to have
Skydive Agents/Analyzer set up with the proper probes
by DevStack.

For a single node setup adding the following lines to your local.conf file
should be enough.

{% highlight shell %}
enable_plugin skydive https://github.com/skydive-project/skydive.git

enable_service skydive-agent skydive-analyzer
{% endhighlight %}

The plugin accepts the following parameters:

{% highlight shell %}
# Address on which skydive analyzer process listens for connections.
# Must be in ip:port format
#SKYDIVE_ANALYZER_LISTEN=

# Configure the skydive analyzer with the etcd server address
# IP_ADDRESS:12379
#SKYDIVE_ANALYZER_ETCD=

# Inform the agent about the address on which analyzers are listening
# Must be in ip:port format
#SKYDIVE_ANALYZERS=

# ip:port address on which skydive agent listens for connections.
#SKYDIVE_AGENT_LISTEN=

# The path for the generated skydive configuration file
#SKYDIVE_CONFIG_FILE=

# List of agent probes to be used by the agent
# Ex: netns netlink ovsdb
#SKYDIVE_AGENT_PROBES=

# Remote port for ovsdb server.
#SKYDIVE_OVSDB_REMOTE_PORT=6640

# Set the default log level, default: INFO
#SKYDIVE_LOGLEVEL=DEBUG

# List of public interfaces for the agents to register in fabric
#SKYDIVE_PUBLIC_INTERFACES="devstack1/eth0 devstack2/eth1"
{% endhighlight %}

### The classical two nodes deployment

Inside the `devstack` folder of the
[Skydive sources](https://github.com/skydive-project/skydive/tree/master/devstack)
there are two local.conf files that can be used in order to deployment two Devstack with Skydive.
The first file will install a full Devstack with Skydive analyzer and agent. The second
one will install a compute Devstack with only the skydive agent.

For Skydive to create a TOR object that links both Devstack, add the following
line to your local.conf file :

{% highlight shell %}
SKYDIVE_PUBLIC_INTERFACES="devstack1/eth0 devstack2/eth1"
{% endhighlight %}

where `devstack1` and `devstack2` are the hostnames of the two nodes followed
by their respective public interface.

Skydive will be set with the probes for OpenvSwitch and Neutron. It will be set
to use Keystone as authentication mechanism, so the credentials will be the same
than the admin.

## Client & WebUI

### Client

Skydive client can be used to interact with Skydive Analyzer and Agents.
Running it without any command will return all the commands available.

{% highlight shell %}
skydive client
Usage:
  skydive client [command]

Available Commands:
  alert         Manage alerts
  capture       Manage captures
  inject-packet Inject packets
  pcap          Import flows from PCAP file
  query         Issue Gremlin queries
  shell         Shell Command Line Interface
  status        Show analyzer status
  topology      Request on topology [deprecated: use 'client query' instead]
  user-metadata Manage user metadata

Flags:
      --analyzer string   analyzer address
  -h, --help              help for client
      --password string   password auth parameter
      --username string   username auth parameter

Global Flags:
  -c, --conf stringArray        location of Skydive configuration files, default try loading /etc/skydive/skydive.yml if exist
  -b, --config-backend string   configuration backend (defaults to file) (default "file")

Use "skydive client [command] --help" for more information about a command.
{% endhighlight %}

Specifying the subcommand will give the usage of the subcommand.

{% highlight shell %}
$ skydive client capture
Manage captures

Usage:
  skydive client capture [command]

Available Commands:
  create      Create capture
  delete      Delete capture
  get         Display capture
  list        List captures

Flags:
  -h, --help   help for capture

Global Flags:
      --analyzer string         analyzer address
  -c, --conf stringArray        location of Skydive configuration files, default try loading /etc/skydive/skydive.yml if exist
  -b, --config-backend string   configuration backend (defaults to file) (default "file")
      --password string         password auth parameter
      --username string         username auth parameter

Use "skydive client capture [command] --help" for more information about a command.
{% endhighlight %}

If an authentication mechanism is defined in the configuration file the username
and password parameter have to be used for each command. Environment variables
`SKYDIVE_USERNAME` and `SKYDIVE_PASSWORD` can be used as default value for the
username/password command line parameters.

Skydive uses the Gremlin traversal language as a topology request language.
Requests on the topology can be done as following :

{% highlight shell %}
$ skydive client query "G.V().Has('Name', 'br-int', 'Type' ,'ovsbridge')"
[
  {
    "Host": "pc48.home",
    "ID": "1e4fc503-312c-4e4f-4bf5-26263ce82e0b",
    "Metadata": {
      "Name": "br-int",
      "Type": "ovsbridge",
      "UUID": "c80cf5a7-998b-49ca-b2b2-7a1d050facc8"
    }
  }
]
{% endhighlight %}

Refer to the
<a href="/documentation/cli">Gremlin section</a> for further
explanations about the syntax and the functions available.

### WebUI

To access to the WebUI of agents or analyzer :

{% highlight shell %}
http://<address>:<port>
{% endhighlight %}

### History and Datastore

Skydive can keep track of all the modifications of the topology and flows in a datastore. Skydive supports two backends: 
[Elasticsearch](https://www.elastic.co/products/elasticsearch) and [OrientDB](https://orientdb.com/).

In order to activate the history we need first to define the storage in the configuration 
[file](https://github.com/skydive-project/skydive/blob/master/etc/skydive.yml.default#L369)

`Elasticsearch` example:

```
storage:
  myelasticsearch:
    driver: elasticsearch
    host: 127.0.0.1:9200
```

Then we need to use it as topology 
[backend](https://github.com/skydive-project/skydive/blob/master/etc/skydive.yml.default#L81):

```
  topology:
    backend: myelasticsearch
```

and as Flow [backend](https://github.com/skydive-project/skydive/blob/master/etc/skydive.yml.default#L74)

```
  flow:
    backend: myelasticsearch
```

### Grafana

In order to use the Grafana datasource plugin we need to use a data store, please see the upper section.

A docker image with the datasource plugin available can be use as following:

```
docker run -d --name=grafana -p 3000:3000 skydive/skydive-grafana-datasource
```

For further information, please see the 
[plugin README](https://github.com/skydive-project/skydive-grafana-datasource)
