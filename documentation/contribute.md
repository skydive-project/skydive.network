---
title: Documentation
section: Contribute
layout: doc
---

Contributions are more than welcome. 

You can follow the 
[how to build](http://skydive.network/documentation/build)
documentation to get your development environment. We provide
a fully functional development Vagrant box. With it you will have
everything needed to compile Skydive and to run the unit and the
functional tests.

## Development box

A Vagrant box with all the required dependencies to compile Skydive and run its
testsuite is
<a href="https://app.vagrantup.com/skydive/boxes/skydive-dev" target="_blank">
  available.
</a>

Supported platforms:

* Windows hosts
* OS X hosts
* Linux distributions

Install one of the following hypervisors:

* VirtualBox (v5.2.10 or v5.2.12)
* libvirt

Install vagrant (v2.1.1).

Then download the run the `Skydive Development Box`:

{% highlight shell %}
git clone https://github.com/skydive-project/skydive.git
cd skydive/contrib/dev
vagrant up
vagrant ssh
{% endhighlight %}

Note: As the Skydive sources will be shared between the host and the guest you need to ensure that
the firewall allows the NFS traffic on the host.

## How to compile and test

Once logged you just need to go to the Skydive source folder and to run `make install`

{% highlight shell %}
cd src/github.com/skydive-project/skydive
make install
{% endhighlight %}

Now we have a fresh compiled Skydive binary we can start it using the `standalone` mode.

{% highlight shell %}
SKYDIVE_ANALYZER_LISTEN=0.0.0.0:8082 SKYDIVE_ETCD_DATA_DIR=/tmp sudo -E ~/bin/skydive allinone
{% endhighlight %}

API and WeUI are availble from the host at this address : http://192.168.10.10:8082

# How to test WebUI modifications

The Skydive WebUI is embedded within the binary. In order to not have to re-compile the
binary each time you want to test modifications, Skydive has to be compiled 
using the `debug` target. To build the target in `debug` mode, set the `DEBUG`
environment variable to `true`

{% highlight shell %}
export DEBUG=true
make install
{% endhighlight %}

Using this mode, assets will be read directly from the disk instead of using the bundled ones. 
A browser refresh will be enough to see any modification.

The WebUI sources are in the `statics` folder of the Skydive sources.
