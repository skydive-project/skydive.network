---
title: Documentation
section: Python library
layout: doc
---

## Installation

Skydive client library can be installed using pip.

{% highlight shell %}
pip install skydive-client
{% endhighlight %}

The Library provides two kinds of client. One is the REST client which will be used
to make topology/flow requests. The WebSocket client will be used in order to
interact with the Graph engine of Skydive. With it, we will be able to listen
graph events or to inject graph elements in the Skydive topology.

## Rest API

The following example requests nodes named eth0 and print some details 
about the first returned interface.

{% highlight python %}
from skydive.rest.client import RESTClient

restclient = RESTClient("localhost:8082", username="admin", password="password")
nodes = restclient.lookup_nodes("G.V().Has('Name', 'eth0')")

print(nodes[0].metadata["Name"])
print(nodes[0].metadata["Type"])
print(nodes[0].metadata["MAC"])
print(nodes[0].metadata["IPV4"])
{% endhighlight %}

To request any kind of object from the topology API we can use the `lookup` method.
The following example requests all the `ICMPv4` flows.

{% highlight python %}
from skydive.rest.client import RESTClient

restclient = RESTClient("38.145.32.195:8082", username="admin", password="password")
flows = restclient.lookup("G.Flows().Has('Application', 'ICMPv4')")
{% endhighlight %}

Captures can be created, listed, deleted thanks to the `capture_create`, `capture_list`, `capture_delete`
methods:

{% highlight python %}
from skydive.rest.client import RESTClient

restclient = RESTClient("38.145.32.195:8082", username="admin", password="password")
restclient.capture_create("G.V().Has('Name', 'eth0', 'Type', 'device')")
{% endhighlight %}


## WebSocket API

Thanks to this API we can subcribe to the graph event bus of Skydive being able to
see all the modifications of the topology.

The following example shows how to add two nodes connected with a link.

{% highlight python %}
import sys

from skydive.graph import Node, Edge
from skydive.rest.client import RESTClient

from skydive.websocket.client import NodeAddedMsgType, EdgeAddedMsgType
from skydive.websocket.client import WSClient, WSClientDefaultProtocol, WSMessage


class WSClientInjectProtocol(WSClientDefaultProtocol):

    def onOpen(self):
        print("WebSocket connection open.")

        # create the first node
        node = Node("NODE1", "myhost",
                    metadata={"Name": "The node 1", "Type": "device"})
        msg = WSMessage("Graph", NodeAddedMsgType, node)
        self.sendWSMessage(msg)

        # create the seconde node
        node = Node("NODE2", "myhost",
                    metadata={"Name": "The node 2", "Type": "device"})
        msg = WSMessage("Graph", NodeAddedMsgType, node)
        self.sendWSMessage(msg)

        # create the link between the 2 nodes
        edge = Edge("EDGE1", "myhost", "NODE1", "NODE2",
                    metadata={"RelationType": "layer2", "Type": "fabric"})
        msg = WSMessage("Graph", EdgeAddedMsgType, edge)
        self.sendWSMessage(msg)

        print("Success!")

    def onClose(self, wasClean, code, reason):
        self.stop()


wsclient = WSClient("myhost", "ws://localhost:8082/ws/publisher",
                    protocol=WSClientInjectProtocol, persistent=True)
wsclient.connect()
wsclient.start()
{% endhighlight %}

## Command line

A command line tool comes with the client library which allows to listen for graph events.

{% highlight shell %}
skydive-ws-client --analyzer localhost:8082 --username admin --password password \
  listen

DEBUG:skydive.websocket.client:transport, protocol: <_SelectorSocketTransport fd=3 read=polling write=<idle, bufsize=0>>, <skydive.websocket.client.WSClientDebugProtocol object at 0x7f9d1f2e9710>
DEBUG:skydive.websocket.client:Connected: tcp:::1:8082
DEBUG:skydive.websocket.client:WebSocket connection opened.


DEBUG:skydive.websocket.client:Text message received: {"Namespace":"Graph","Type":"NodeUpdated","UUID":"3ffc351a-1120-43bb-6bbb-e9a5f99977aa","Status":200,"Obj":{"ID":"e517e855-d4ba-4428-55cc-45fbf2b20f38","Metadata":{"Driver":"","EncapType":"loopback","IPV4":["127.0.0.1/8"],"IPV6":["::1/128"],"IfIndex":1,"LastUpdateMetric":{"Last":1530284389687,"RxBytes":56247,"RxPackets":489,"Start":1530284359687,"TxBytes":56247,"TxPackets":489},"MAC":"","MTU":65536,"Metric":{"Last":1530284389687,"RxBytes":1718864452,"RxPackets":4277534,"TxBytes":1718864452,"TxPackets":4277534},"Name":"lo","Neighbors":[{"IP":"0.0.0.0","IfIndex":1,"MAC":"00:00:00:00:00:00","State":["NUD_NOARP"]},{"IP":"::1","IfIndex":1,"MAC":"00:00:00:00:00:00","State":["NUD_NOARP"]}],"RoutingTable":[{"Id":255,"Routes":[{"Nexthops":[{"IfIndex":1}],"Prefix":"127.0.0.0/32","Protocol":2},{"Nexthops":[{"IfIndex":1}],"Prefix":"127.0.0.0/8","Protocol":2},{"Nexthops":[{"IfIndex":1}],"Prefix":"127.0.0.1/32","Protocol":2},{"Nexthops":[{"IfIndex":1}],"Prefix":"127.255.255.255/32","Protocol":2},{"Nexthops":[{"IfIndex":1}],"Prefix":"::1/128","Protocol":2}],"Src":"127.0.0.1"},{"Id":254,"Routes":[{"Nexthops":[{"IfIndex":1,"Priority":256}],"Prefix":"::1/128","Protocol":2}]}],"State":"UP","Type":"device"},"Host":"localhost.localdomain","CreatedAt":1530284329660,"UpdatedAt":1530284389687,"Revision":3}}
{% endhighlight %}

To get the full graph when connecting :

{% highlight shell %}
skydive-ws-client --analyzer localhost:8082 --username admin --password password \
  listen --sync-request
{% endhighlight %}

### Dump / inject

It is also possible to inject graph elements. It can be useful to re-inject a dump
from one Skydive instance to an other one.

The dump is easy to achieve, a simple `curl` will do the job.

{% highlight shell %}
curl -o /tmp/skydive.json http://localhost:8082/api/topology
{% endhighlight %}

To re-inject the following command line can be use.

{% highlight shell %}
skydive-ws-client --analyzer localhost:8082 --username admin --password password \
  add /tmp/skydive.dump 
{% endhighlight %}