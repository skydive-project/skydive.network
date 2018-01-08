---
title: First steps
section: 4. Packet injector
layout: first-steps
---

<p>This part will be an introduction of the Packet Generator/Injector feature that Skydive provide. We'll keep playing with the WebUI which allow us to
get started smoothly. The goal of the Packet Generator is to forge packets and to inject them into an interface of the topology. Using this feature with the packet capture allow
us to verify if the traffic is forwarded as expected.</p>
<h2>Packet injection</h2>
<p>
  Starting with the lab created in the previous part, two network namespaces with a `ShortestPath` capture, we are going to use
  the Packet Injector to generate pings instead of the "ping" utility.
</p>

<p>
  For that purpose, on the WebUI, we just need to click on the `Generator` tab on the right panel. Here you can select the type of
  the packet that will be generated. For now we will use the default one. Then you have to select the source, the destination and the
  number of packets you want to generate. By default Skydive will use the first IP address available of the node, but you can change it.
  There are some other options like the delay between two packets or the payload length but for
  the demo we will use the default values.
</p>

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/packet-injector-1.webm"></video>
<p>

<p>
  As in the previous part, we can click on a node of the path to check if we see our generated packets.
</p>

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/packet-injector-2.webm"></video>
<p>

<h2>Let's drop some packets</h2>

<p>
  Now we can simulate an issue somewhere in order to see how we can detect where the issue occurs. For that we are going to add an iptables rule in order to
  drop traffic for specify TCP port. Let say the port 4567.
</p>

<p class="code">
  <code>
    sudo iptables -t filter -A FORWARD -i br0 -o br0 -m physdev --physdev-is-bridged -j DROP
  </code>
</p>

Now we can use the generator or anything else like Netcat to generate some UDP or TCP packets, then we can refresh the flow view in order to check where our
packets have been seen. It will confirm that the packets were dropped around the bridge `br0`.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/packet-injector-3.webm"></video>
<p>

We saw how easy it is to target where a packet is dropped. Of course this can be achieved with the Skydive client too. In the following part we will
see how to use the client to get the same result.

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-3.html"><i class="fa fa-chevron-left" aria-hidden="true"> 3. Traffic capture, multiple interfaces</i></a>
  </p>
</div>
