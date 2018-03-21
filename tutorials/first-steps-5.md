---
title: First steps
section: 5. Multi-node and tunneling
layout: first-steps
---

<p>In the previous parts we saw the basics thanks to a single node sandbox. In this part we will use a multi-nodes deployment and we will see how Skydive can analyze an encapsulated traffic. There are different handy ways to deploy a multi-node environment with Skydive, here we will use the `Vagrantfile` which comes with the Skydive repository.</p>

<h2>One Vagrantfile, three nodes</h2>
<p>
  First be sure that you have enough memory, the deployment requires a bit less than 5G of memory. As said in the introduction, the `Vagrantfile` comes with the sources of Skydive, so we just need to clone the project. Go to the `contrib` folder and start vagrant.
</p>

{% highlight shell %}
git clone https://github.com/skydive-project/skydive
cd skydive/contrib/vagrant
vagrant up
{% endhighlight %}

3 hosts were deployed : 1 analyzer and 2 agents. The analyzer WebUI is available at `http://192.168.50.10:8082`.

With the topology expanded, we will get something like this :

<p>
  {% lightbox /assets/images/first-steps/multi-nodes-1.png --thumb="/assets/images/first-steps/multi-nodes-1.png" --data="multi-nodes-1" --alt="Capture" %}
</p>

What the `Vagrantfile` did is simple. It deployed the Skydive binary on every host, using the default config file which can be found
<a href="https://raw.githubusercontent.com/skydive-project/skydive/master/etc/skydive.yml.default">here</a>. The config is stored in
`/etc/skydive/skydive.yml`.

For the agents, only the `analyzers` list is set to specify the analyzer address.

{% highlight yaml %}
analyzers:
- 192.168.50.10:8082
{% endhighlight %}

The analyzer configuration is a bit more complex. Elasticsearch is specified as backend for the flows :

{% highlight yaml %}
analyzer:
  storage:
    backend: elasticsearch
{% endhighlight %}

and the topology :

{% highlight yaml %}
graph:
  backend: elasticsearch
{% endhighlight %}

The way the agents are connected together is also described in the config file. We are not going to detail here the syntax but basically it creates a `TOR1` node and two port nodes `TOR1_PORT1`, `TOR1_PORT2` connected to the `eth1` interface of the host agents.

{% highlight yaml %}
analyzer:
  topology:
    fabric:
    - TOR1[Name=tor1] -> TOR1_PORT1[Name=port1, MTU=1500]
    - TOR1_PORT1 -> *[Type=host, Name=agent1]/eth1
    - TOR1[Name=tor1] -> TOR1_PORT2[Name=port2, MTU=1500]
    - TOR1_PORT2 -> *[Type=host, Name=agent2]/eth1
{% endhighlight %}

<h2>Packet journey within a tunnel</h2>

Now we have a properly working multi-node deployment, we can create a topology with tunnels. For that purpose we use Open vSwitch with GRE tunnels. The following command lines executed on the agent1 will create a network namespace connected to a bridge and will add a GRE interface :

{% highlight shell %}
vagrant ssh agent1

sudo ip netns add ns1
sudo ip link add ns1-eth0 type veth peer name eth0 netns ns1
sudo ip link set ns1-eth0 up
sudo ip netns exec ns1 ip link set eth0 up
sudo ip netns exec ns1 ip address add 10.0.0.1/24 dev eth0
sudo ovs-vsctl add-br br0
sudo ovs-vsctl add-port br0 ns1-eth0
sudo ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.50.30
{% endhighlight %}

Same thing on the second agent host with the following command lines :

{% highlight shell %}
vagrant ssh agent2

sudo ip netns add ns2
sudo ip link add ns2-eth0 type veth peer name eth0 netns ns2
sudo ip link set ns2-eth0 up
sudo ip netns exec ns2 ip link set eth0 up
sudo ip netns exec ns2 ip address add 10.0.0.2/24 dev eth0
sudo ovs-vsctl add-br br0
sudo ovs-vsctl add-port br0 ns2-eth0
sudo ovs-vsctl add-port br0 gre0 -- set interface gre0 type=gre options:remote_ip=192.168.50.20
{% endhighlight %}

<p>
  Once executed, Skydive should have detected the two network namespaces with all the links and should look like this.
</p>

<p>
  {% lightbox /assets/images/first-steps/multi-nodes-2.png --thumb="/assets/images/first-steps/multi-nodes-2.png" --data="multi-nodes-2" --alt="Capture" %}
</p>

Now we can create 3 captures, one on a the `eth1` interface of an agent host, and one other on each Open vSwitch bridge.
Once the captures are created, we can use the packet injector to generate pings between the two network namespaces.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/multi-nodes-1.webm"></video>
</p>

By selecting the `eth1` interface, we can check the flows and see that we have our `ICMP` packets and a `GRE` flow. We see that Skydive is analyzing inner packets and thanks to that we can confirm that a packet has been seen even within a tunnel. We can verify the relation between the inner packet and the outer packet by checking the field `ParentUUID` of the inner packet. This field references the `UUID` of the flow of the tunnel.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/multi-nodes-2.webm"></video>
</p>

Skydive currently supports tunneling protocol like GRE, VXLAN, GENEVE, MPLS/GRE, MPLS/UDP and supports multiple levels of encapsulation.

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-4.html"><i class="fa fa-chevron-left" aria-hidden="true"> 4. Packet injector</i></a>
  </p>
  <p style="float:right">
    <a href="/tutorials/first-steps-6.html">6. API/CLI tour <i class="fa fa-chevron-right" aria-hidden="true"></i></a>
  </p>
</div>
