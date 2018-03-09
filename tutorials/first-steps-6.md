---
title: First steps
section: 6. API/CLI tour
layout: first-steps
---

<p>In the previous parts we discovered Skydive through its WebUI. In this part
we are now going to use the Skydive command line to reproduce what we did previously.</p>

<h2>Skydive client, Gremlin requests</h2>

In order to have an easy fully functional Skydive multi-node deployment we will use the
`Vagrant` environment as previously.

{% highlight shell %}
git clone https://github.com/skydive-project/skydive
cd skydive/contrib/vagrant
vagrant up
{% endhighlight %}

<p>
  Following the previous part you should have the following topology in the WebUI.
</p>

<p>
  {% lightbox /assets/images/first-steps/multi-nodes-1.png --thumb="/assets/images/first-steps/multi-nodes-1.png" --data="multi-nodes-1" --alt="Capture" %}
</p>

Now that we have our lab deployed we can connect to the `analyzer` node in order to
use the `Skydive` binary to request the Skydive API.

{% highlight shell %}
vagrant ssh analyzer1
{% endhighlight %}

Skydive uses the `Gremlin` graph traversal language as its query language.
The following request returns all the `eth0` interfaces present in to topology
using the `JSON` format.

{% highlight shell %}
export SKYDIVE_ANALYZERS=192.168.50.10:8082

skydive client query "G.V().Has('Name', 'eth0')"
{% endhighlight %}

We can obtain the full topology in a `dot` format with the following request :

{% highlight shell %}
skydive client query "G" --format dot
{% endhighlight %}

Which gives once rendered the following image:

<p>
{% lightbox /assets/images/first-steps/dot.png --thumb="/assets/images/first-steps/dot.png" --data="dot" --alt="Dot output" %}
</p>

In order to get the state of the `eth0` belonging to specific host, we can use this :

{% highlight shell %}
skydive client query "G.V().Has('Name', 'agent1').Out().Has('Name', 'eth0')"
{% endhighlight %}

This has to be read as "select the node with the name `agent1` then select node
with the name `eth0`"

A more exhaustive documentation of the Skydive Gremlin language is available
<a href="http://skydive-project.github.io/skydive/api/gremlin/">here</a>

<h2>Capture request</h2>

As seen before it is easy to select interfaces of the topology and starting a traffic
capture is as simple as writing a select request.

To start a traffic capture on all the `eth0` interface we just have to reuse
our first `Gremlin` request like this :

{% highlight shell %}
skydive client capture create --gremlin "G.V().Has('Name', 'eth0')"
{% endhighlight %}

In order to check that the capture definition matched two interfaces we can
just list them.

{% highlight shell %}
skydive client capture list
{
  "2a1988b1-0d4b-4df5-5b90-9196e6a8d837": {
    "UUID": "2a1988b1-0d4b-4df5-5b90-9196e6a8d837",
    "GremlinQuery": "G.V().Has('Name', 'eth0')",
    "Count": 2,
    "ExtraTCPMetric": false,
    "SocketInfo": false
  }
}
{% endhighlight %}

The count field indicates how many traffic capture have been started.
Another way to check is to write a request retrieving the interfaces on which
the capture have been started.

{% highlight shell %}
skydive client query "G.V().Has('Capture.ID', '2a1988b1-0d4b-4df5-5b90-9196e6a8d837').Count()"
2

skydive client query "G.V().Has('Capture.ID', '2a1988b1-0d4b-4df5-5b90-9196e6a8d837').Values('Name')"
[
	"eth0",
	"eth0"
]

skydive client query "G.V().Has('Capture.ID', '2a1988b1-0d4b-4df5-5b90-9196e6a8d837').Values('Host')"
[
	"agent2",
	"agent1"
]
{% endhighlight %}

<h2>Packet injection</h2>

We are going to use the command to generate ICMPv4 packets. Packets will be injected from one `eth0` towards the other `eth0`.
First we need to get the `IDs` of the interfaces.

{% highlight shell %}
skydive client query "G.V().Has('Name', 'eth0').Values('ID')"
[
	"a5f2fd0d-1f30-46a9-75ff-6ba4c82426a1",
	"99d511f5-5088-4c0d-67db-e45af33cb87c"
]
{% endhighlight %}

Now the packet injection request :

{% highlight shell %}
 skydive client inject-packet create \
   --src "G.V('a5f2fd0d-1f30-46a9-75ff-6ba4c82426a1')" \
   --dst "G.V('99d511f5-5088-4c0d-67db-e45af33cb87c')" \
   --count 5
{% endhighlight %}

And we can finally get the flows. We will get 2 flows as we started two captures so
one ECHO/REPLY flow per capture.

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'ICMPv4')"
{% endhighlight %}

We can just select a specific field using the `Values` `Gremlin` step.

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'ICMPv4').Values('Network')"
[
	{
		"Protocol": "IPV4",
		"A": "192.168.121.112",
		"B": "192.168.121.217",
		"ID": 0
	},
	{
		"Protocol": "IPV4",
		"A": "192.168.121.112",
		"B": "192.168.121.217",
		"ID": 0
	}
]
{% endhighlight %}

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'ICMPv4').Values('Metric')"
[
	{
		"ABPackets": 5,
		"ABBytes": 210,
		"BAPackets": 5,
		"BABytes": 210,
		"Start": 1520545797027,
		"Last": 1520545797027
	},
	{
		"ABPackets": 5,
		"ABBytes": 300,
		"BAPackets": 5,
		"BABytes": 210,
		"Start": 1520545796988,
		"Last": 1520545796988
	}
]
{% endhighlight %}

Flows have a unique `UUID` per capture but also have a `TrackingID` which is the same across multiple captures.

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'ICMPv4').Values('TrackingID')"
[
	"b53dafa2e10d19d5c1598e0bc723b7eb5bc3655a",
	"b53dafa2e10d19d5c1598e0bc723b7eb5bc3655a"
]
{% endhighlight %}

The `Gremlin` step `Dedup` de-duplicates flow according to the `TrackingID`.

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'ICMPv4').Dedup()"
[
	{
		"UUID": "bad1803ce7caa70e7835356afdccacaa8e89b185",
		"LayersPath": "Ethernet/IPv4/ICMPv4",
		"Application": "ICMPv4",
		"Link": {
			"Protocol": "ETHERNET",
			"A": "52:54:00:cc:e2:ef",
			"B": "52:54:00:43:8c:a3",
			"ID": 0
		},
		"Network": {
			"Protocol": "IPV4",
			"A": "192.168.121.112",
			"B": "192.168.121.217",
			"ID": 0
		},
		"ICMP": {
			"Type": "ECHO",
			"Code": 0,
			"ID": 0
		},
		"Metric": {
			"ABPackets": 5,
			"ABBytes": 210,
			"BAPackets": 5,
			"BABytes": 210,
			"Start": 1520545797027,
			"Last": 1520545797027
		},
		"Start": 1520545797027,
		"Last": 1520545797027,
		"RTT": 208265,
		"TrackingID": "b53dafa2e10d19d5c1598e0bc723b7eb5bc3655a",
		"L3TrackingID": "52e47eab007caeb74cfba34f713242f939a11ce7",
		"ParentUUID": "",
		"NodeTID": "50037073-7862-5234-4996-e58cc067c69c",
		"BNodeTID": "50037073-7862-5234-4996-e58cc067c69c",
		"RawPacketsCaptured": 0
	}
]
{% endhighlight %}

We can also specify the capture from where we want to get the flow.

{% highlight shell %}
skydive client query "G.V('50037073-7862-5234-4996-e58cc067c69c').Flows().Has('Application', 'ICMPv4')"
[
	{
		"UUID": "bad1803ce7caa70e7835356afdccacaa8e89b185",
		"LayersPath": "Ethernet/IPv4/ICMPv4",
		"Application": "ICMPv4",
		"Link": {
			"Protocol": "ETHERNET",
			"A": "52:54:00:cc:e2:ef",
			"B": "52:54:00:43:8c:a3",
			"ID": 0
		},
		"Network": {
			"Protocol": "IPV4",
			"A": "192.168.121.112",
			"B": "192.168.121.217",
			"ID": 0
		},
		"ICMP": {
			"Type": "ECHO",
			"Code": 0,
			"ID": 0
		},
		"Metric": {
			"ABPackets": 5,
			"ABBytes": 210,
			"BAPackets": 5,
			"BABytes": 210,
			"Start": 1520545797027,
			"Last": 1520545797027
		},
		"Start": 1520545797027,
		"Last": 1520545797027,
		"RTT": 208265,
		"TrackingID": "b53dafa2e10d19d5c1598e0bc723b7eb5bc3655a",
		"L3TrackingID": "52e47eab007caeb74cfba34f713242f939a11ce7",
		"ParentUUID": "",
		"NodeTID": "50037073-7862-5234-4996-e58cc067c69c",
		"BNodeTID": "50037073-7862-5234-4996-e58cc067c69c",
		"RawPacketsCaptured": 0
	}
]
{% endhighlight %}

It is also possible to start from a flow to get all the captured interfaces
where `ICMPv4` flows have been seen.

```
skydive client query "G.Flows().Has('Application', 'ICMPv4').CaptureNode().Values('ID')"
[
	"a5f2fd0d-1f30-46a9-75ff-6ba4c82426a1",
	"99d511f5-5088-4c0d-67db-e45af33cb87c"
]
```

In order to get only one specific flow we need to use its `TrackingID` as this ID is
the same across multiple capture points.

```
skydive client query "G.Flows().Has('TrackingID', 'b53dafa2e10d19d5c1598e0bc723b7eb5bc3655a').CaptureNode().Values('ID')"
[
	"a5f2fd0d-1f30-46a9-75ff-6ba4c82426a1",
	"99d511f5-5088-4c0d-67db-e45af33cb87c"
]
```

This post was just an introduction to the Skydive API and query language. The same language
is leveraged by some other features that Skydive brings, like `alerting`. A more complete
documentation is available <a href="http://skydive-project.github.io/">here</a>

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-5.html"><i class="fa fa-chevron-left" aria-hidden="true"> 5. Multi-node and tunneling</i></a>
  </p>
  <p style="float:right">
    <a href="/tutorials/first-steps-7.html">7. Advanced capture options <i class="fa fa-chevron-right" aria-hidden="true"></i></a>
  </p>
</div>
