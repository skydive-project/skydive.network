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
* Go >= 1.9
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

Ports to be used need to be uncommented or added.
