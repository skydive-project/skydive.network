---
title: First steps
section: 3. Traffic capture, multiple interfaces
layout: first-steps
---

<p>In this third part we will see how to create a capture matching multiple interfaces and how to follow a flow along a path.</p>
<h2>Two network namespaces lab</h2>
<p>
  To keep the demonstration simple we are just going to use a tiny shell script that will create two network namespaces interconnected through
  a bridges thanks to veth interfaces.
</p>

{% highlight shell %}
sudo ip netns add ns1
sudo ip link add ns1-eth0 type veth peer name eth0 netns ns1
sudo ip link set ns1-eth0 up
sudo ip netns exec ns1 ip link set eth0 up
sudo ip netns exec ns1 ip address add 10.0.0.1/24 dev eth0

sudo ip netns add ns2
sudo ip link add ns2-eth0 type veth peer name eth0 netns ns2
sudo ip link set ns2-eth0 up
sudo ip netns exec ns2 ip link set eth0 up
sudo ip netns exec ns2 ip address add 10.0.0.2/24 dev eth0

sudo ip link add name br0 type bridge
sudo ip link set dev br0 up
sudo ip link set dev ns1-eth0 master br0
sudo ip link set dev ns2-eth0 master br0
{% endhighlight %}

<p>
  Once executed, Skydive should have detected the two namespaces with all the links and should look like this.
</p>

<p>
  <img src="/assets/images/first-steps/capture-2ns-1.png"/>
</p>

<h2>Shortest path capture</h2>

<p>
  After having expanded the two namespaces, double clicking on them or thanks to the left botton expand button, we can start a capture using the same
  process as for capturing one interface except here we are going to select the `eth0` of the first namespace as first interface and the `eth0` of the
  second namespace as second interface.
</p>

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/capture-3.webm"></video>
<p>

<h2>Journey of a packet</h2>
<p>
  Let's generate a bit of traffic. Skydive embeds a packet generator but here we are just going to use a simple ping as the packet generator will be addressed in another
  part.
</p>

{% highlight shell %}
sudo ip netns exec ns1 ping 10.0.0.2
{% endhighlight %}

<p>
  We can now check if we see ICMPv4 packets just by clicking on one captured interface in order to get the flows on the right panel. In order to see
  where ICMPv4 packets have been see we can place the mouse over the flow, the interfaces involved in the packet forwarding will be highlighted.
</p>

<p>
  <img src="/assets/images/first-steps/capture-2ns-2.png"/>
</p>

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-2.html"><i class="fa fa-chevron-left" aria-hidden="true"> 2. Traffic capture</i></a>
  </p>
  <p style="float:right">
    <a href="/tutorials/first-steps-4.html">4. Packet injector <i class="fa fa-chevron-right" aria-hidden="true"></i></a>
  </p>
</div>
