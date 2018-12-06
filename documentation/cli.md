---
title: Documentation
section: Command Line Interface
layout: doc
---

Skydive uses the Gremlin traversal language as a topology and flow request language.

## Topology requests

Requests on the topology can be done as following :

{% highlight shell %}
$ skydive client query "G.V().Has('Name', 'br-int', 'Type' ,'ovsbridge')"
[
  {
    "Host": "pc48.home",
    "ID": "1e4fc503-312c-4e4f-4bf5-26263ce82e0b",
    "Metadata": {
      "Name": "br-int",
      "Type": "ovsbridge",
      "UUID": "c80cf5a7-998b-49ca-b2b2-7a1d050facc8"
    }
  }
]
{% endhighlight %}

Refer to the
<a href="/documentation/api-gremlin">Gremlin section</a> for further
explanations about the syntax and the functions available.

## Flow requests

A typical Gremlin request on Flows will return a JSON version of the Flow
structure.

{% highlight shell %}
skydive client query "G.Flows().Has('Application', 'DNS').Limit(1)"
[
	{
		"UUID": "c40232c35c791ccf",
		"LayersPath": "Ethernet/IPv4/UDP/DNS",
		"Application": "DNS",
		"Link": {
			"Protocol": "ETHERNET",
			"A": "48:45:20:17:cf:0f",
			"B": "d8:6c:e9:98:54:ec",
			"ID": 0
		},
		"Network": {
			"Protocol": "IPV4",
			"A": "192.168.1.21",
			"B": "192.168.1.254",
			"ID": 0
		},
		"Transport": {
			"Protocol": "UDP",
			"A": "51413",
			"B": "53",
			"ID": 0
		},
		"Metric": {
			"ABPackets": 1,
			"ABBytes": 75,
			"BAPackets": 1,
			"BABytes": 112,
			"Start": 1526046091335,
			"Last": 1526046091345
		},
		"IPMetric": {
			"Fragments": 0,
			"FragmentErrors": 0
		},
		"Start": 1526046091335,
		"Last": 1526046091345,
		"RTT": 9803721,
		"TrackingID": "2c3429d2e093c415",
		"L3TrackingID": "cd65a3cef30f705b",
		"ParentUUID": "",
		"NodeTID": "42286bd1-dff4-5428-7b97-903f32820542",
		"RawPacketsCaptured": 0
	}
]
{% endhighlight %}

The Flow Schema is described in a
<a href="https://github.com/skydive-project/skydive/blob/master/flow/flow.proto" target="_blank">
  protobuf file
</a>.

## Flow capture

The following command starts a capture on all docker0 interfaces :

{% highlight shell %}
skydive client capture create --gremlin "G.V().Has('Name', 'docker0')" --type pcap
{
  "UUID": "76de5697-106a-4f50-7455-47c2fa7a964f",
  "GremlinQuery": "G.V().Has('Name', 'docker0')"
}
{% endhighlight %}

The Gremlin expression is continuously evaluated which
means that it is possible to define a capture on nodes that do not exist yet.
It useful when you want to start a capture on all OpenvSwitch whatever the
number of Skydive agents you will start.

While starting the capture, you can specify the capture name,
capture description and capture type optionally.

At this time, the following capture types are supported :

* `ovssflow`, for interfaces managed by OpenvSwitch such as OVS bridges
* `afpacket`, for interfaces suchs as Linux bridges, veth, devices, ...
* `pcap`, same as `afpacket`
* `sFlow`, Socket reading sFlow frames
* `dpdk`, for interfaces managed by DPDK
* `pcapsocket`. This capture type allows you to inject traffic from a PCAP file.
* `ovsmirror`, leverages OpenvSwitch port mirroring.
* `eBPF`, flow capture within kernel

### Delete a capture

{% highlight shell %}
skydive client capture delete <capture UUID>
{% endhighlight %}

### PCAP files

If the flow probe `pcapsocket` is enabled, you can create captures with the
type `pcapsocket`. Skydive will create a TCP socket where you can copy PCAP
files (using `nc` for instance). Traffic injected into this socket will have
its capture point set to the selected node. The TCP socket address can be
retrieved using the `PCAPSocket` attribute of the node or using the
`PCAPSocket` attribute of the capture.

## Packet injector

Skydive provides a Packet injector, which is helpful to verify the successful packet flow between two network devices by injecting a packet from one device and capture the same in the other device.

### Create packet injector

To use the packet injector we need to provide the below parameters,

* Source node, needs to be expressed in gremlin query format.
* Destination node, needs to be expressed in gremlin query format, optional if dstIP and dstMAC given.
* Source IP of the packet, optional if source node given.
* Source MAC of the packet, optional if source node given.
* Destination IP of the packet, optional if destination node given.
* Destination MAC of the packet, optional if destination node given.
* Type of packet. currently ICMP, TCP and UDP are supported.
* Number of packets to be generated, default is 1.
* IMCP ID, used only for ICMP type packets.
* Interval, the delay between two packets in milliseconds.
* Payload
* Source port, used only for TCP and UDP packets, if not given generates one randomly.
* Destination port, used only for TCP and UDP packets, if not given generates one randomly.

{% highlight shell %}
skydive client inject-packet create [flags]

Flags:
      --count int        number of packets to be generated (default 1)
      --dst string       destination node gremlin expression
      --dstIP string     destination node IP
      --dstMAC string    destination node MAC
      --dstPort int      destination port for TCP packet
      --id int           ICMP identification
      --interval int     wait interval milliseconds between sending each packet (default 1000)
      --payload string   payload
      --src string       source node gremlin expression (mandatory)
      --srcIP string     source node IP
      --srcMAC string    source node MAC
      --srcPort int      source port for TCP packet
      --type string      packet type: icmp4, icmp6, tcp4, tcp6, udp4 and udp6 (default "icmp4")
{% endhighlight %}

### Example

{% highlight shell %}
skydive client inject-packet create \
  --src="G.V().Has('TID', 'b0acebfb-9cd0-5de0-787f-366fcccc6651')" \
  --dst="G.V().Has('TID', 'c94a12fd-f159-5228-7c05-e49b3c2bbb04')" \
  --type="icmp4" --count=50 --interval=5000

{
  "UUID": "5b269b65-df92-42c6-4e69-4f8aa30f6110",
  "Src": "G.V().Has('TID', 'b0acebfb-9cd0-5de0-787f-366fcccc6651')",
  "Dst": "G.V().Has('TID', 'c94a12fd-f159-5228-7c05-e49b3c2bbb04')",
  "SrcIP": "",
  "DstIP": "",
  "SrcMAC": "",
  "DstMAC": "",
  "SrcPort": 0,
  "DstPort": 0,
  "Type": "icmp4",
  "Payload": "",
  "TrackingID": "e66c4b372e312be791238099538d8dda1949836b",
  "ICMPID": 0,
  "Count": 50,
  "Interval": 5000,
  "StartTime": "0001-01-01T00:00:00Z"
}
{% endhighlight %}

### Delete packet injection

Deleting the active packet injector can be done by using the `delete` sub-command with the UUID of the injector.

UUID of the injection will be returned as a response of `create`

### Example

{% highlight shell %}
$ skydive client inject-packet delete 5b269b65-df92-42c6-4e69-4f8aa30f6110
{% endhighlight %}

## Alerting

Skydive allows you to create alerts, based on queries on both topology graph
and flows.

### Alert evaluation

An alert can be specified through a [Gremlin](api-gremlin) query or a
JavaScript expression. The alert will be triggered if it returns:

* true
* a non empty string
* a number different from zero
* a non empty array
* a non empty map

The alert is triggered only if the evaluation result differs from the previous evaluation.

Gremlin example:

With the following command, the alert is triggered as soon as the Gremlin query
returns something new (i.e. a new node returned by the query, an updated metadata in the returned nodes...).

{% highlight shell %}
skydive client alert create --expression "G.V().Has('Name', 'eth0', 'State', 'DOWN')"
{
  "UUID": "185c49ba-341d-41a0-6f96-f3224140b2fa",
  "Expression": "G.V().Has('Name', 'eth0', 'State', 'DOWN')",
  "CreateTime": "2016-12-29T13:29:05.273620179+01:00"
}
{% endhighlight %}

JavaScript example:

If you prefer to trigger the alert only when the state of one of the node returned by a gremlin query goes down,
you can use the following JavaScript expression :

{% highlight shell %}
skydive client alert create \
  --expression "states = Gremlin(\"G.V().Has('Name', 'br-int-lb').Values('State')\"); result = false; for (var i = 0; i < states.length; i++){ if (states[i] == 'DOWN'){ result = true; break;}} result;"
{% endhighlight %}

The alert will be triggered every time the result of the JavaScript expression changes.

With the following command, the alert is triggered when the bandwidth of the flow bypass 1Mbps.
It gets trigerred again the next time the bandwidth bypass this threshold.

{% highlight shell %}
skydive client alert create \
  --expression "Gremlin(\"G.Flows().Has('Network.A', '192.168.0.1').Metrics().Sum()\").ABBytes > 1*1024*1024" \
  --trigger "duration:10s"
{
  "UUID": "331b5590-c45d-4723-55f5-0087eef899eb",
  "Expression": "Gremlin(\"G.Flows().Has('Network.A', '192.168.0.1').Metrics().Sum()\").ABBytes > 1*1024*1024",
  "Trigger": "duration:10s",
  "CreateTime": "2016-12-29T13:29:05.197612381+01:00"
}
{% endhighlight %}

### Fields

* `Name`, the alert name (optional)
* `Description`, a description for the alert (optional)
* `Expression`, a Gremlin query or JavaScript expression
* `Action`, URL to trigger. Can be a [local file](cli#webhook) or a [WebHook](cli#script)
* `Trigger`, event that triggers the alert evaluation. Periodic alerts can be
   specified with `duration:5s`, for an alert that will be evaluated every 5 seconds.

### Notifications

When an alert is triggered, all the WebSocket clients will be notified with a
message of type `Alert` with a JSON object with the attributes:

* `UUID`, ID of the triggered alert
* `Timestamp`, timestamp of trigger
* `ReasonData`, the result of the alert evaluation. If `expression` is a
  Gremlin query, it will be the result of the query. If `expression` is a
  JavaScript statement, it will be the result of the evaluation of this
  statement.

In addition to the WebSocket message, an alert can trigger different kind of
actions.

### Webhook

A POST request is issued with the JSON message as payload.

### Script

A local file (prefixed by file://) to execute a script. It receives the JSON
message through stdin

## Topology Rules

Skydive allows you to create and delete Nodes, update node metadata and create and delete Edges with the help of Topology rules API.
It provides two seperate rule for node and edge
* `node-rule`
* `edge-rule`

### Node Rules

With `node-rule` you can create and delete nodes and update node metadata.

`node-rule` contain following fields
* `action`, action: create or update
* `description`, rule description (optional)
* `name`, rule name (optional)
* `node-name`, node name only for create node
* `node-type`, node type only for create node
* `metadata`, node metadata, key value pairs. 'k1=v1, k2=v2'
* `query`, gremlin query only for update

To create a new node, you have to specify the `Action` as `create` and provide the node name and node type.

### Example

{% highlight shell %}
skydive client node-rule create \
  --action="create" \
  --node-name="node1" \
  --node-type="fabric" \
  --metadata="key1=value1, key2=value2"

{
  "UUID": "fcbc566a-14a1-4b39-490d-13f6e0cd4a51",
  "Name": "",
  "Description": "",
  "Metadata": {
    "Name": "node1",
    "Type": "fabric",
    "key1": "value1",
    "key2": "value2"
  },
  "Action": "create",
  "Query": ""
}
{% endhighlight %}

To update a node metadata, you have to specify the `action` as `update` and query to select nodes.

### Example

{% highlight shell %}
skydive client node-rule create \
  --action="update" \
  --metadata="newkey1=newValue1, newkey2=newValue2"
  --query="G.V().Has('Name', 'eth0')"

{
  "UUID": "76bd84cc-d447-4cc8-522e-e35c80353c83",
  "Name": "",
  "Description": "",
  "Metadata": {
    "newkey1": "newValue1",
    "newkey2": "newValue2"
  },
  "Action": "update",
  "Query": "G.V().Has('Name', 'eth0')"
}
{% endhighlight %}

### Delete node rule

Deleting the node rule can be done by using the `delete` sub-command with the UUID of the node rule.

UUID of the node rule will be returned as a response of `create`

### Example

{% highlight shell %}
skydive client node-rule delete 76bd84cc-d447-4cc8-522e-e35c80353c83
{% endhighlight %}

By deleting the done rule, it will delete the node from the graph.

### Edge Rules

With `edge-rule` you can create ans delete edges.

`edge-rule` contain following fields
* `name`, the rule name
* `description`, the rule description
* `src`, source node gremlin query
* `dst`, destination node gremlin query
* `relationtype`, relation type of the link (layer2, ownership and both)
* `metadata`, edge metadata, key value pairs. 'k1=v1, k2=v2'

To create a edge rule, you have to provide the source and destination nodes and the relation type of the edge

### Example

{% highlight shell %}
skydive client edge-rule create \
  --src="G.V().Has('Name', 'node1')" \
  --dst="G.V().Has('Name', 'node2')" \
  --relationtype="layer2" \
  --metadata="key=value"

{
  "UUID": "c7e65252-85bf-494b-76d1-6840a819571f",
  "Name": "",
  "Description": "",
  "Src": "G.V().Has('Name', 'node1')",
  "Dst": "G.V().Has('Name', 'node2')",
  "Metadata": {
    "RelationType": "layer2",
    "key": "value"
  }
}
{% endhighlight %}

### Delete edge rule
Deleting the edge rule can be done by using the `delete` sub-command with the UUID of the edge rule.

UUID of the edge rule will be returned as a response of `create`

### Example

{% highlight shell %}
skydive client edge-rule delete c7e65252-85bf-494b-76d1-6840a819571f
{% endhighlight %}
