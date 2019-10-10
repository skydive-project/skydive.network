---
title: Documentation
section: How to build
layout: doc
---

Skydive relies on two main components:

* skydive agent, has to be started on each node where the topology and flows
  informations will be captured
* skydive analyzer, the node collecting data captured by the agents

## Dependencies

### Minimal
* Go >= 1.11
* libpcap (dev)
* libxml2 (dev)
* libvirt (dev)
* protoc >= 3.0
* protobuf
* npm

## Installation

### Dependencies

Below how to install the minimal required dependencies for Fedora and Ubuntu.

#### Fedora

{% highlight shell %}
sudo dnf install -y git make patch golang findutils protobuf-c-compiler protobuf-devel \
  npm libpcap-devel libxml2-devel libvirt-devel
{% endhighlight %}

#### Ubuntu

{% highlight shell %}
sudo apt-get install -y git make patch golang findutils protobuf-compiler libprotobuf-dev \
  npm libpcap0.8-dev libxml2-dev libvirt-dev
{% endhighlight %}

### Build

Create a dedicated GOPATH.

For example:

{% highlight shell %}
mkdir $HOME/go
export GOPATH=$HOME/go
export PATH=$GOPATH/bin:$PATH
{% endhighlight %}

then get and build Skydive

{% highlight shell %}
mkdir -p $GOPATH/src/github.com/skydive-project
git clone https://github.com/skydive-project/skydive.git \
  $GOPATH/src/github.com/skydive-project/skydive
cd $GOPATH/src/github.com/skydive-project/skydive
make
{% endhighlight %}

### eBPF support

Skydive can leverage `eBPF` programs for topology and flow capture. This provides
a lightweight solution for retrieving topology information such as process socket information
and for packet processing within the kernel.

Some extra dependencies are required :
  * llvm
  * clang
  * kernel-headers / linux-libc-dev
  * bcc / bcc-devel

To enable the eBPF support :

{% highlight shell %}
make install WITH_EBPF=true
{% endhighlight %}

### DPDK flow capture support

Skydive support flow capture from DPDK NICs. This allows starting flow capture on-demand
on DPDK interface like any other interface.

Some extra dependencies are required :
  * numactl-devel
  * kernel-devel

To enable the DPDK support :

{% highlight shell %}
make install WITH_DPDK=true
{% endhighlight %}

The DPDK probe requires some configuration adjustments. Below the DPDK configuration
section :

{% highlight shell %}
dpdk:
  # DPDK port listening flows from
  ports:
    # - 0
    # - 1

  # nb workers per port
  # workers: 4

  # debug message every n seconds
  # debug: 1
{% endhighlight %}

Ports to be used need to be un-commented or added.

## Third Parties

### Collectd plugin

`Skydive Project` provides a `Collectd` plugin which allows to expose `Collectd` metrics to the `Skydive` topology. The plugin has to be
started on the same host than `Skydive` agents and will update the `Host` node of the `Skydive` topology.

In order to compile the `Skydive Collectd plugin` one need to have the [Collectd sources](https://github.com/collectd/collectd).

{% highlight shell %}
export COLLECTD_SRC=/tmp/collectd

git clone https://github.com/collectd/collectd $COLLECTD_SRC

mkdir -p $GOPATH/src/github.com/skydive-project
git clone https://github.com/skydive-project/skydive-collectd-plugin.git \
  $GOPATH/src/github.com/skydive-project/skydive-collectd-plugin
cd $GOPATH/src/github.com/skydive-project/skydive-collectd-plugin

make
{% endhighlight %}

This will generate a shared object (`skydive.so`) that can be placed in the collectd plugin folder.

In order to use it, it has to be copied in the collectd plugin folder. In order to configure it the following section
has to be added to the `collectd` config file.

{% highlight shell %}
LoadPlugin skydive
<Plugin skydive>
    Address "127.0.0.1:8081"
    Username ""
    Password ""
</Plugin>
{% endhighlight %}

* `Address` is the `Skydive` agent address
* `Username` `Skydive` cluster authentication user name
* `Password` `Skydive` cluster authentication password

Example:

In order to get memory metrics from `Collectd`:

{% highlight shell %}
LoadPlugin memory

LoadPlugin skydive
<Plugin skydive>
    Address "127.0.0.1:8081"
    Username ""
    Password ""
</Plugin>
{% endhighlight %}

All the `Collectd` metrics are currently reported in a `Collectd` sub-key of the `Host` topology node.

{% highlight shell %}
skydive client query 'G.V().HasKey("Collectd").Values("Collectd")
{% endhighlight %}
