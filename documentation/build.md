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

* Go >= 1.9
* Elasticsearch >= 2.0
* libpcap
* libxml2
* protoc >= 3.0
* llvm
* clang
* kernel-headers / linux-libc-dev
* bcc / bcc-devel
* npm

## Installation

Make sure you have a working Go environment.
<a href="http://golang.org/doc/install.html" target="_blank">
 See the install instructions
</a>

{% highlight shell %}
mkdir -p $GOPATH/src/github.com/skydive-project
git clone https://github.com/skydive-project/skydive.git $GOPATH/src/github.com/skydive-project/skydive
cd $GOPATH/src/github.com/skydive-project/skydive
make install
{% endhighlight %}

### eBPF support

Skydive can leverage `eBPF` programs for topology and flow capture. This provides
a lightweight solution for retrieving topology information such as process socket information
and for packet processing within the kernel.

To enable the eBPF support :

{% highlight shell %}
make install WITH_EBPF=true
{% endhighlight %}

### DPDK flow capture support

Skydive support flow capture from DPDK NICs. This allows starting flow capture on-demand
on DPDK interface like any other interface.

To enable the DPDK support :

{% highlight shell %}
make install WITH_DPDK=true
{% endhighlight %}

Some extra dependencies are required :
 * numactl-devel
 * kernel-devel

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
