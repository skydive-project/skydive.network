---
title: First steps
section: 7. Advanced capture options
layout: first-steps
---

<p>Until now we saw how to create simple capture using either the WebUI or the
command line. This part will be a tour of the "advanced" capture options.</p>

<h2>BPF filtering</h2>

First let start our lab with the usual `Vagrant` deployment.

{% highlight shell %}
git clone https://github.com/skydive-project/skydive
cd skydive/contrib/vagrant
vagrant up
{% endhighlight %}

While we create a capture we can decide to capture everything or just a part of
the traffic leveraging `BPF` filtering via the
<a href="https://www.tcpdump.org/manpages/pcap-filter.7.html">PCAP filter syntax</a>.

Filtering ICMPv4 packets with the WebUI :

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/advanced-captures-1.webm"></video>
</p>

Or with the command line :

{% highlight shell %}
export SKYDIVE_ANALYZERS=192.168.50.10:8082

skydive client capture create --gremlin "G.V().Has('Name', 'eth0')" --bpf "icmp"
{% endhighlight %}

<h2>Capture types</h2>

Skydive supports multiple ways to capture packets. By default the most adapted type
is chosen automatically by Skydive. For example if you start a capture on a Open vSwitch
bridge, the `sFlow` type will be used.

Here the list of the currently supported capture types :

* AFPACKET : MMap'd AF_PACKET socket (default).
* PCAP : Packet Capture library based.
* PCAP Socket : open a TCP port accepting PCAP file format.
  Useful to inject already captured traffic to Skydive.
* sFlow : implement a sFlow agent. It opens a UDP port reading `sFlow` frames.
  Useful to send `sFlow` traffic from external resources to Skydive.
* eBPF : in Kernel lightweight capture solution.
* OvsMirror : leverages Open vSwitch port mirroring.

Capture type has to be specified during the capture creation.

{% highlight shell %}
skydive client capture create --gremlin "G.V().Has('Name', 'eth0')" --type pcap
{% endhighlight %}

<h2>Keep original packets</h2>

While Skydive analyzes the packets to build flows, keeping them in flow tables,
it is possible to keep the original packets attached to flows. This allows us to
download them as PCAP file. That way Skydive acts as a distributed `tcpdump`.

The following video shows how to start a such capture and how to retrieve the
PCAP file.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/advanced-captures-2.webm"></video>
</p>

Clicking on the download icon we get the file which opened with `Wireshark` gives :

<p>
  <a href="/assets/images/first-steps/advanced-captures-1.png" data-lightbox="WebUI-1" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/advanced-captures-1.png"/>
  </a>
</p>

We can of course use the command line to create the capture, here limiting to 5 raw packets
per flow.

{% highlight shell %}
skydive client capture create --gremlin "G.V().Has('Name', 'eth0')" --rawpacket-limit 5
{% endhighlight %}

To get the `PCAP` file we just need to use the `Gremlin` step `RawPackets` specifying the
output format.

{% highlight shell %}
skydive client query "G.At('-1s', 1000).Flows().Has('Application', 'ICMPv4').RawPackets()" \
 --format pcap > icmp.pcap
{% endhighlight %}

This searches flows with the `ICMPv4` protocol since the last 1000 seconds
exporting their `RawPackets` in the icmp.pcap file.

{% highlight shell %}
tcpdump -qns 0 -r icmp.pcap
reading from file icmp.pcap, link-type EN10MB (Ethernet)
15:43:02.626000 IP 192.168.50.20 > 192.168.50.30: ICMP echo request, id 0, seq 0, length 8
15:43:02.626000 IP 192.168.50.30 > 192.168.50.20: ICMP echo reply, id 0, seq 0, length 8
15:43:02.626000 IP 192.168.50.20 > 192.168.50.30: ICMP echo request, id 0, seq 0, length 8
15:43:02.626000 IP 192.168.50.30 > 192.168.50.20: ICMP echo reply, id 0, seq 0, length 8
{% endhighlight %}

It gaves us 4 packets, 2 per capture from 2 different hosts.

`BPF`, `RawPackets` captures and the query language makes Skydive a really powerful
distributed troubleshooting tool.

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-6.html"><i class="fa fa-chevron-left" aria-hidden="true"> 6. API/CLI tour</i></a>
  </p>
</div>
