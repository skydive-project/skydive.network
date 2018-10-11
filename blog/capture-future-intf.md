---
title: Capture for interfaces that don't exist (yet)
layout: blog-post
author: Sylvain Afchain
date: 09/10/2018
---

I have been reached out a couple of times about a feature request which was something like :

"I would like to capture the very first packets, like DHCP, of a VM that is about to boot. It would be great to add a mechanism to start a capture for a future interface."

So yes it will be nice, and in fact it is already possible since the beginning of Skydive as it was one of the use cases that we wanted to address. Let's see how to achieve this.

## First packets in the life of a new interface

As an example I'll use an `OpenStack` environment. You can have a look at the [Getting started](/documentation/getting-started#openstackdevstack) section to know how to have 
`Skydive` deployed as part of `Devstack`.

Once deployed we get:

<p>
  <a href="/assets/images/blog/capture-future-1.png" data-lightbox="WebUI-1" data-title="Skydive capture">
    <img src="/assets/images/blog/capture-future-1.png"/>
  </a>
</p>

Now it is time to create our capture. I'll use the Web UI but the CLI could be used as well.

In order to do so, just click on the `Create` button of the `Capture` tab. Then instead of using the node selection I'll use
the `Gremlin Expression` mode with the following expression.

{% highlight shell %}
G.V().HasKey('ExtID.vm-id')
{% endhighlight %}

The gremlin expression has to be read like : 

"Capture all the interface having the attribute ExtID.vm-id"

In an OpenStack environment, VMs interfaces are plugged into an `OpenvSwitch` bridge and the `OpenvSwitch/Neutron` probes of Skydive reports the "VM ID" as `ExtID.vm-id` metadata attribute.

Now that the capture is created, we can see the `Captures` list our capture with a "Warning" icon. This icon indicates that the capture definition currently doesn't match any interface and consequently no packet capture has been started (yet).

<p>
  <a href="/assets/images/blog/capture-future-2.png" data-lightbox="WebUI-2" data-title="Skydive capture">
    <img src="/assets/images/blog/capture-future-2.png"/>
  </a>
</p>

Now we just have to boot a VM to see `Skydive` triggering packet capture automatically.

As soon as the interface is created `Skydive` starts the packet capture, then you can click on the interface to see the very first packets. On the following capture you'll see `DCHP` packets, `ICMP` packets and of course `TCP` connections towards the
metadata service. We can conclude that there is no connectivity issue between the VM and the `OpenStack` services. 

<p>
  <a href="/assets/images/blog/capture-future-3.png" data-lightbox="WebUI-3" data-title="Skydive capture">
    <img src="/assets/images/blog/capture-future-3.png"/>
  </a>
</p>

Of course we could script a validation using the API or the command line.

## conclusion

In a very short term we will introduce a way to leverage this feature and the [Workflow](/blog/introduction-to-workflows.html) feature to automatically check
the connectivity for newly created interface. This will bring the ability to do complex network health checks. Stay tuned !