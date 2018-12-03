---
title: Documentation
section: Gremlin query language
layout: doc
---

Skydive uses a subset of the
[Gremlin language](https://en.wikipedia.org/wiki/Gremlin_\(programming_language\))
as query language for topology and flow requests.

A Gremlin expression is a chain of steps that are evaluated from left to right.
In the context of Skydive nodes stand for interfaces, ports, bridges,
namespaces, etc. Links stand for any kind of relation between two nodes,
ownership(host, netns, ovsbridge, containers), layer2, etc.

The following expression will return all the OpenvSwitch ports belonging to
an OpenvSwitch bridge named `br-int`.

{% highlight shell %}
G.V().Has('Name', 'br-int', 'Type', 'ovsbridge').Out()

[
  {
    "Host": "test",
    "ID": "feae10c1-240e-48e0-4a13-c608ffd15700",
    "Metadata": {
      "Name": "vm2-eth0",
      "Type": "ovsport",
      "UUID": "f68f3593-68c4-4778-b47f-0ef291654fcf"
    }
  },
  {
    "Host": "test",
    "ID": "ca909ccf-203d-457d-70b8-06fe308221ef",
    "Metadata": {
      "Name": "br-int",
      "Type": "ovsport",
      "UUID": "e5b47010-f479-4def-b2d0-d55f5dbf7dad"
    }
  }
]
{% endhighlight %}

The following expression will return all the k8s Pods belonging to the
k8s NameSpace named `my-namespace`.

{% highlight shell %}
G.V().Has('Docker.Labels.io.kubernetes.pod.namespace' ,'my-namespace').Descendants()
{% endhighlight %}

The query has to be read as :

1. `G` returns the topology Graph
2. `V` step returns all the nodes belonging the Graph
3. `Has` step returns only the node with the given metadata attributes
4. `Out` step returns outgoing nodes

## Traversal steps

The Skydive implements a subset of the Gremlin language steps and adds
"network analysis" specific steps.

### V

V step returns the nodes belonging to the graph.

{% highlight shell %}
G.V()
{% endhighlight %}

A node ID can be passed to the V step which will return the corresponding node.

{% highlight shell %}
G.V('ca909ccf-203d-457d-70b8-06fe308221ef')
{% endhighlight %}

### E

E step returns the edges belonging to the graph.

{% highlight shell %}
G.E()
{% endhighlight %}

A edge ID can be passed to the E step which will return the corresponding edge.

{% highlight shell %}
G.E('c8aeb26f-0962-4c46-b700-a12dfe720af1')
{% endhighlight %}

### Has

`Has` step filters out the nodes that don't match the given metadata list. `Has`
can be applied either on nodes or edges.

{% highlight shell %}
G.V().Has('Name', 'test', 'Type', 'netns')
{% endhighlight %}

### HasKey

`HasKey` step filters out the nodes that don't match the given key list. `HasKey`
can be applied either on nodes or edges.

{% highlight shell %}
G.V().HasKey('Name')
{% endhighlight %}

### In/Out/Both

`In/Out` steps returns either incoming, outgoing or neighbor nodes of
previously selected nodes.

{% highlight shell %}
G.V().Has('Name', 'br-int', 'Type', 'ovsbridge').Out()
G.V().Has('Name', 'br-int', 'Type', 'ovsbridge').In()
G.V().Has('Name', 'br-int', 'Type', 'ovsbridge').Both()
{% endhighlight %}

Filters can be applied to these steps in order to select only the nodes
corresponding to the given metadata. In that case the step will act as a couple
of steps `Out/Has` for example.

{% highlight shell %}
G.V().Has('Name', 'br-int', 'Type', 'ovsbridge').Out('Name', 'intf1')
{% endhighlight %}

### InE/OutE/BothE

`InE/OutE/BothE` steps returns the incoming/ougoing links.

{% highlight shell %}
G.V().Has('Name', 'test', 'Type', 'netns').InE()
G.V().Has('Name', 'test', 'Type', 'netns').OutE()
G.V().Has('Name', 'test', 'Type', 'netns').BothE()
{% endhighlight %}

Like for the `In/Out/Both` steps metadata list can be passed directly as
parameters in order to filter links.

### InV/OutV

`InV/OutV` steps returns incoming, outgoing nodes attached to the previously
selected links.

{% highlight shell %}
G.V().OutE().Has('Type', 'layer2').InV()
{% endhighlight %}

### Dedup

`Dedup` removes duplicated nodes/links or flows. `Dedup` can take a parameter
in order to specify the field used for the deduplication.

{% highlight shell %}
G.V().Out().Both().Dedup()
G.V().Out().Both().Dedup('Type')
G.Flows().Dedup('NodeTID')
{% endhighlight %}

### Count

`Count` returns the number of elements retrieved by the previous step.

{% highlight shell %}
G.V().Count()
{% endhighlight %}

### Values

`Values` returns the property value of elements retrieved by the previous step.

{% highlight shell %}
G.V().Values('Name')
{% endhighlight %}

### Keys

`Keys` returns the list of properties of the elements retrieved by the previous step.

{% highlight shell %}
G.V().Keys()
{% endhighlight %}

### Sum

`Sum` returns sum of elements, named 'Name', retrieved by the previous step.
When attribute 'Name' exists, must be integer type.

{% highlight shell %}
G.V().Sum('Name')
{% endhighlight %}

### Limit

`Limit` limits the number of elements returned.

{% highlight shell %}
g.Flows().Limit(1)
{% endhighlight %}

### ShortestPathTo

`ShortestPathTo` step returns the shortest path to node matching the given
`Metadata` predicate. This step returns a list of all the nodes traversed.

{% highlight shell %}
G.V().Has('Type', 'netns').ShortestPathTo(Metadata('Type', 'host'))

[
  [
    {
      "Host": "test",
      "ID": "5221d3c3-3180-4a64-5337-f2f66b83ddd6",
      "Metadata": {
        "Name": "vm1",
        "Path": "/var/run/netns/vm1",
        "Type": "netns"
      }
    },
    {
      "Host": "test",
      "ID": "test",
      "Metadata": {
        "Name": "pc48.home",
        "Type": "host"
      }
    }
  ]
]
{% endhighlight %}

It is possible to filter the link traversed according to the given `Metadata`
predicate as a second parameter.

{% highlight shell %}
G.V().Has('Type', 'netns').ShortestPathTo(Metadata('Type', 'host'), Metadata('RelationType', 'layer2'))
{% endhighlight %}

### SubGraph

`SubGraph` step returns a new Graph based on the previous steps. Step V or E can
be used to walk trough this new Graph.

{% highlight shell %}
G.E().Has('RelationType', 'layer2').SubGraph().V().Has('Name', 'eth0')
{% endhighlight %}

{% highlight shell %}
G.V().Has('Type', 'veth').SubGraph().E()
{% endhighlight %}

### GraphPath

`GraphPath` step returns a path string corresponding to the reverse path
from the nodes to the host node they belong to.

{% highlight shell %}
G.V().Has('Type', 'netns').GraphPath()

[
  "test[Type=host]/vm1[Type=netns]"
]
{% endhighlight %}

The format of the path returned is the following:
`node_name[Type=node_type]/.../node_name[Type=node_type]``

### At

`At` allows to set the time context of the Gremlin request. It means that
we can contextualize a request to a specific point of time therefore being
able to see how was the graph in the past.
Supported formats for time argument are :

* Timestamp
* RFC1123 format
* [Go Duration format](https://golang.org/pkg/time/#ParseDuration)

{% highlight shell %}
G.At(1479899809).V()
G.At('-1m').V()
G.At('Sun, 06 Nov 2016 08:49:37 GMT').V()
{% endhighlight %}

`At` takes also an optional duration parameter which allows to specify a
period of time in second for the lookup. This is useful especially when retrieving
metrics. See [`Metrics` step](/api/gremlin#metrics-step) for more information.

{% highlight shell %}
G.At('-1m', 500).V()
G.At('-1m', 3600).Flows()
{% endhighlight %}

### Flows

Flows step returns flows of nodes where a capture has been started or of nodes
where the packets are coming from or going to.
The following Gremlin query returns the flows from the node `br-int` where
an sFlow capture has been started.
See the [client section](/getting-started/client/#flow-captures)
in order to know how to start a capture from a Gremlin query.

{% highlight shell %}
G.V().Has('Name', 'br-int').Flows()
{% endhighlight %}

### Flows In/Out

From a flow step it is possible to get the node from where the packets are
coming or the node where packets are going to. Node steps are of course
applicable after `In/Out` flow steps.

{% highlight shell %}
G.V().Has('Name', 'br-int').Flows().In()
G.V().Has('Name', 'br-int').Flows().Out()
{% endhighlight %}

### Flows Has

`Has` step filters out the flows that don't match the given attributes list.

{% highlight shell %}
G.Flows().Has('Network.A', '192.168.0.1')
{% endhighlight %}

Key can be any attributes of the Flow data structure :

* `UUID`
* `TrackingID`
* `NodeTID`
* `ANodeTID`
* `BNodeTID`
* `LayersPath`
* `Application`
* `Link`
* `Link.A`
* `Link.B`
* `Link.Protocol`
* `Network`
* `Network.A`
* `Network.B`
* `Network.Protocol`
* `Transport`
* `Transport.A`
* `Transport.B`
* `Transport.Protocol`
* `Metric.ABBytes`
* `Metric.BABytes`
* `Metric.ABPackets`
* `Metric.BAPackets`
* `Start`
* `Last`

Lt, Lte, Gt, Gte predicates can be used on numerical fields.
See [Flow Schema](/api/flows/) for further explanations.

Link, Network and Transport keys shall be matched with any of A or B by using OR operator.

### Flows Sort

`Sort` step sorts flows by the given field and requested order.
By default, the flows are in ascending order by their `Last` field.
`ASC` and `DESC` predicates can be used to specify ascending and descending order respectively.

{% highlight shell %}
G.Flows().Sort()
G.Flows().Sort("Metric.ABPackets")
G.Flows().Sort(DESC, "Metric.ABPackets")
{% endhighlight %}

### Flows Dedup

`Dedup` step de-duplicates flows having the same TrackingID.

{% highlight shell %}
G.Flows().Dedup()
{% endhighlight %}

### Flows Group step

`Group` step returns the flows matching the same argument field (TrackingID by default).

{% highlight shell %}
G.Flows().Group("Network.Protocol")
{% endhighlight %}

### Metrics

`Metrics` returns arrays of metrics of a set of flows or interfaces, grouped by
the flows UUIDs or Node IDs.

For flow metrics :

{% highlight shell %}
G.Flows().Metrics()
[
  {
    "64249da029a25d09668ea4a61b14a02c3d083da0": [
      {
        "ABBytes": 980,
        "ABPackets": 10,
        "BABytes": 980,
        "BAPackets": 10,
        "Last": 1479899789,
        "Start": 1479899779
      }
  }
]
{% endhighlight %}

and for interface metrics :

{% highlight shell %}
G.V().Metrics()
[
  {
    "fc2a6103-599e-4821-4c87-c8224bd0e84e": [
    {
      "Collisions": 0,
      "Last": 1489161773820,
      "Multicast": 0,
      "RxBytes": 7880,
      "RxCompressed": 0,
      "RxCrcErrors": 0,
      "RxDropped": 0,
      "RxErrors": 0,
      "RxFifoErrors": 0,
      "RxFrameErrors": 0,
      "RxLengthErrors": 0,
      "RxMissedErrors": 0,
      "RxOverErrors": 0,
      "RxPackets": 10,
      "Start": 1489161768820,
      "TxAbortedErrors": 0,
      "TxBytes": 7880,
      "TxCarrierErrors": 0,
      "TxCompressed": 0,
      "TxDropped": 0,
      "TxErrors": 0,
      "TxFifoErrors": 0,
      "TxHeartbeatErrors": 0,
      "TxPackets": 10,
      "TxWindowErrors": 0
    }

  }
]
{% endhighlight %}

## Predicates

Predicates which can be used with `Has`, `In*`, `Out*` steps :

* `NE`, matches graph elements for which metadata don't match specified values

{% highlight shell %}
G.V().Has('Type', NE('ovsbridge'))
{% endhighlight %}

* `Within`, matches graph elements for which metadata values match one member of
  the given array.

{% highlight shell %}
G.V().Has('Type', Within('ovsbridge', 'ovsport'))
{% endhighlight %}

* `Without`, matches graph elements for which metadata values don't match any of
  the members of the given array.

{% highlight shell %}
G.V().Has('Type', Without('ovsbridge', 'ovsport'))
{% endhighlight %}

* `Regex`, matches graph elements for which metadata matches the given regular
  expression.

{% highlight shell %}
G.V().Has('Name', Regex('tap.*'))
{% endhighlight %}

Please note that the Regex step is always using anchors, ^ and $ don't have to
be provided in the expression.
