---
title: Documentation
section: API
layout: doc
---

## REST API

### Topology/Flow request

{% highlight shell %}
POST /api/topology HTTP/1.1
Content-Type: application/json

{
  "GremlinQuery":"G.V()"
}
{% endhighlight %}

{% highlight shell %}
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

[
  {
    "Host": "localhost.localdomain",
    "ID": "d6759df3-d4e0-408b-64d3-c82ea6c9aeda",
    "Metadata": {
      "Name": "vm2",
      "Path": "/var/run/netns/vm2",
      "TID": "7daa39fe-92f7-5f9b-51b1-1dddcd41785c",
      "Type": "netns"
    }
  }
]
{% endhighlight %}

{% highlight shell %}
POST /api/topology HTTP/1.1
Content-Type: application/json

{
  "GremlinQuery":"G.Flows().Limit(1)"
}
{% endhighlight %}

{% highlight shell %}
[
  {
    "ANodeTID": "d9d6f8cf-4aa6-4a06-6785-3dc56032ef82",
    "BNodeTID": "488789f9-38be-4eba-704a-79996382de41",
    "LastUpdateMetric": {
      "ABBytes": 490,
      "ABPackets": 5,
      "BABytes": 490,
      "BAPackets": 5,
      "Last": 1477572621,
      "Start": 1477572616
    },
    "LayersPath": "Ethernet/IPv4/ICMPv4",
    "Link": {
      "A": "02:48:4f:c4:40:99",
      "B": "e2:d0:f0:61:e7:81",
      "Protocol": "ETHERNET"
    },
    "Metric": {
      "ABBytes": 1666,
      "ABPackets": 17,
      "BABytes": 1568,
      "BAPackets": 16,
      "Last": 1477572622,
      "Start": 1477572606
    },
    "Network": {
      "A": "192.168.0.1",
      "B": "192.168.0.2",
      "Protocol": "IPV4"
    },
    "NodeTID": "488789f9-38be-4eba-704a-79996382de41",
    "TrackingID": "f745fb1f59298a1773e35827adfa42dab4f469f9",
    "UUID": "ee29fc47f425d7a2e6de9379b0131f64a70fc991"
  }
]
{% endhighlight %}

### Capture

To create capture :

{% highlight shell %}
POST /api/capture HTTP/1.1
Content-Type: application/json

{
  "GremlinQuery":"g.V().Has('TID', 'de0cba34-5d96-5ce6-698a-dffd2e674f95')"
}
{% endhighlight %}

{% highlight shell %}
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8

{
  "UUID":"e2d9f084-4543-4f7e-6c2c-673f56ae4610",
  "GremlinQuery":"g.V().Has('TID', 'de0cba34-5d96-5ce6-698a-dffd2e674f95')"
}
{% endhighlight %}

To list captures :

{% highlight shell %}
GET /api/capture HTTP/1.1
Content-Type: application/json
{% endhighlight %}

{% highlight shell %}
{
  "104fc114-e153-4f67-692a-60c636ee1597":
  {
    "UUID": "104fc114-e153-4f67-692a-60c636ee1597"
    "GremlinQuery": "G.V().Has('TID', '2108e074-feac-5a3c-60ca-5963e89c4059')"
    "Count": 1
  },
  "e2d9f084-4543-4f7e-6c2c-673f56ae4610":
  {
    "UUID": "e2d9f084-4543-4f7e-6c2c-673f56ae4610"
    "GremlinQuery": "g.V().Has('TID', 'de0cba34-5d96-5ce6-698a-dffd2e674f95')"
    "Count": 1
  }
}
{% endhighlight %}

To delete a capture :

{% highlight shell %}
DELETE /api/capture/7ca73f92-0547-475e-472d-d6e28664a117 HTTP/1.1
Content-Type: application/json
{% endhighlight %}

{% highlight shell %}
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
{% endhighlight %}

## WebSocket API

### Flow stream

Since version `0.21` it is possible to subscribe to analyzers flow stream. This feature can be use
to react on new flow events or to store/send them to an external tool. The flow stream is provided as
a WebSocket endpoint. You can use the `skydive-client` Python library or the Golang one. By default
flows will be sent in JSON, but can also be sent in Protobuf if requested.

The WebSocket endpoint is `/ws/subscriber/flow`

The following example shows how to subscribe with using the Python library, and write the received flows to a file.

First you need to install the `skydive-client` library :

{% highlight shell %}
pip install skydive-client
{% endhighlight %}

And the script itself :

{% highlight shell %}
import json

from skydive.websocket.client import WSClient
from skydive.websocket.client import WSClientDefaultProtocol


class WSLoggerProtocol(WSClientDefaultProtocol):

    def onMessage(self, payload, isBinary):
        msg = json.loads(payload)

        file = self.factory.kwargs["file"]

        for flow in msg["Obj"]:
            file.write(json.dumps(flow))
        print "wrote %d flows" % len(msg["Obj"])


def main():
    file = open("/tmp/flows", "w")

    client = WSClient("MyHost", "ws://127.0.0.1:8082/ws/subscriber/flow",
                      protocol=WSLoggerProtocol,
                      file=file)
    client.connect()
    client.start()


if __name__ == '__main__':
    main()
{% endhighlight %}