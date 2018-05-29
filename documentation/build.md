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

* Go >= 1.8
* Elasticsearch >= 2.0
* libpcap
* libxml2
* protoc >= 3.0
* llvm
* clang
* kernel-headers / linux-libc-dev
* bcc / bcc-devel

## Installation

Make sure you have a working Go environment. [See the install instructions]
(http://golang.org/doc/install.html).

{% highlight shell %}
mkdir -p $GOPATH/src/github.com/skydive-project
git clone https://github.com/skydive-project/skydive.git $GOPATH/src/github.com/skydive-project/skydive
cd $GOPATH/src/github.com/skydive-project/skydive
make install
{% endhighlight %}

## Development box

A Vagrant box with all the required dependencies to compile Skydive and run its
testsuite is
<a href="https://app.vagrantup.com/skydive/boxes/skydive-dev" target="_blank">
  available.
</a>

The image has been successfuly tested on:

* vagrant v2.1.1
* VirtualBox v5.2.10

First install the needed vagrant plugins:

{% highlight shell %}
vagrant install plugin vagrant-openstack-provider
vagrant install plugin vagrant-reload
{% endhighlight %}

Then to use it:

{% highlight shell %}
git clone https://github.com/skydive-project/skydive.git
cd skydive/contrib/dev
vagrant up
vagrant ssh
{% endhighlight %}

The box is available for both `VirtualBox` and `libvirt`.
