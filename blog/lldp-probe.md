---
title: Discover topology using LLDP
layout: blog-post
author: Sylvain Baubeau
date: 09/10/2018
---

We recently added support for [automatic topology discovery using Ansible.](/blog/ansible-library)
During last hackathon, we discussed about adding an LLDP probe in a dynamic way by creating
a LLDP probe in Skydive.

Let's see how it works.

## How to use it
Enabling LLDP in Skydive is very easy, you only need to enable the `lldp` probe in
the agent configuration file (section agent -> topology -> probes). By default, Skydive
will listen for LLDP packets on - almost - all interfaces. You can explicitely
specify the list of the interfaces using the `interfaces` attribute in section `agent -> topology -> lldp`.

When started, the probe will start listening for LLDP traffic on his list of interfaces.
When a LLDP packet is received, Skydive will parse the packet and retrieve LLDP information. It will
then create one node for the `chassis` and an other node for the `port`. It will attach
LLDP information as metadata of both nodes.

<p class="center">
  <a href="/assets/images/blog/lldp.png" data-lightbox="LLDP" data-title="LLDP topology view">
    <img src="/assets/images/blog/lldp.png"/>
  </a>
</p>

## Why not using an external daemon
- The self-contained aspect of Skydive is very important for us. We want to keep the
  ability to just copy the binary and have everything working as much as possible.
- Capturing and analyzing traffic is something Skydive already does obviously :-)
- The library we use for flow dissection [gopacket](http://github.com/google/gopacket)
  has pretty good LLDP support

The probe is pretty small (around 500 lines of codes) and simple. That makes it a nice
example of how to add a Skydive topology probe.

## A minimal probe

A Skydive probe only has to implement the `Probe` interface that defines only 2 methods:
`Start` and `Stop`. So our very first version of our probe could be:

{% highlight golang %}
type Probe struct {
}

func (p *Probe) Start() {
}

func (p *Probe) Stop() {
}

func NewProbe() (*Probe, error) {
  return &Probe{}, nil
}
{% endhighlight %}

We then modify the `agent/probes.go` file to instantiate our probe:
{% highlight golang %}
	case "lldp":
		lldpProbe, err := lldp.NewProbe()
		if err != nil {
			return nil, fmt.Errorf("Failed to initialize LLDP probe: %s", err)
		}
		probes[t] = lldpProbe
{% endhighlight %}

At this point, Skydive should be able to run with this empty probe.

## Probe principles
When starting the probe, some interfaces may not be present yet, or in a wrong state.
So we want to start listen LLDP traffic when they appear or when their state is changed.
The Skydive `netlink` probe will populate the graph with a node when an interface appears and
will update its metadata when its state changed.

Our probe will therefore listen for graph events and consider only the nodes -
matching capable network interfaces - and start capturing LLDP traffic.

A graph listener must implement 6 methods:
- OnNodeAdded(n *graph.Node)
- OnNodeUpdated(n *graph.Node)
- OnNodeDeleted(n *graph.Node)
- OnEdgeAdded(e *graph.Edge)
- OnEdgeUpdated(n *graph.Edge)
- OnEdgeDeleted(e *graph.Edge)

We are not interested in all the events. `DefaultGraphListener` implements an empty version
of every one of them. So we can embed `DefaultGraphListener` in our probe and only
implement the ones we are interested.

{% highlight golang %}
type Probe struct {
	graph.DefaultGraphListener
	...
}
{% endhighlight %}

The events we care about are:
- OnEdgeAdded: when an interface appears, the `netlink` probe will create a node for the interface
               and link it to its owner: the host machine
- OnNodeUpdated: when the state of an interface changes

For both of the events, we decide if we should start a capture. We should start a capture when
- no LLDP capture is running for this interface
- when its first packet layer is Ethernet and it has a MAC address
- when the interface is listed in the configuration file or we are in auto discovery mode

{% highlight golang %}
func (p *Probe) handleNode(n *graph.Node) {
	firstLayerType, _ := probes.GoPacketFirstLayerType(n)
	mac, _ := n.GetFieldString("MAC")
	name, _ := n.GetFieldString("Name")

	if name != "" && mac != "" && firstLayerType == layers.LayerTypeEthernet {
		if active, found := p.interfaceMap[name]; (found || p.autoDiscovery) && !active {
			logging.GetLogger().Infof("Starting LLDP capture on %s", name)
			if err := p.startCapture(name, mac, n); err != nil {
				logging.GetLogger().Error(err)
			}
		}
	}
}
{% endhighlight %}

## Starting LLDP capture

To get the LLDP frames, we capture traffic using AFpacket and read at most
`lldpSnapLen` (which is set at 4096) bytes of the packet. We specify a BPF filter
to only receive LLDP packets.

{% highlight golang %}
	packetProbe, err := probes.NewGoPacketProbe(p.g, n, probes.AFPacket, bpfFilter, lldpSnapLen)
{% endhighlight %}

We can now receive LLDP packets by calling the `Run` method with a callback that
will receive the packet as argument.

{% highlight golang %}
	packetProbe.Run(func(packet gopacket.Packet) {
		p.handlePacket(n, packet)
	}, nil)
{% endhighlight %}

We now need to handle the LLDP packet and enhance the graph with this information.
Every LLDP frame is associated by a chassis and a port. So it makes sense to create
a node for each. LLDP information will be stored inside the metadata.
Let's prepare the metadata for the chassis:

{% highlight golang %}
	lldpLayer := packet.Layer(layers.LayerTypeLinkLayerDiscovery).(*layers.LinkLayerDiscovery)

	chassisMetadata := graph.Metadata{
		"LLDP": map[string]interface{}{
			"ChassisIDType": lldpLayer.ChassisID.Subtype.String(),
		},
		"Type":  "switch",
	}

	var chassisID string
	switch lldpLayer.ChassisID.Subtype {
	case layers.LLDPChassisIDSubTypeMACAddr:
		chassisID = net.HardwareAddr(lldpLayer.ChassisID.ID).String()
	default:
		chassisID = string(lldpLayer.ChassisID.ID)
	}
	common.SetField(chassisMetadata, "LLDP.ChassisID", chassisID)
	common.SetField(chassisMetadata, "Name", chassisID)
{% endhighlight %}

We then retrieve more LLDP info - such as VLANs, Link Aggregation - but we'll
skip this part as it mostly copying fields from the `gopacket` structures
to the metadata map.

If it's the first LLDP packet that we received, a new node for this chassis
should be created. Otherwise we can just retrieve this existing node and
update the metadata with fresh values. To create a node, an ID must be specified
along with the metadata. In this case, we need to carefully generate the ID.

The LLDP probe is executed by the Skydive agent. LLDP information is added to the
agent graph as described above. The agent will forward his graph to a Skydive
analyzer. The analyzer receives the graph of multiple agents. These agents
could be running on machines connected to the same network switch, and therefore
receive LLDP from the same chassis. We need to avoid having multiple nodes
in the Skydive graph corresponding to the same chassis. One way is to generate
the chassis node with an ID that will be the same for all the agents.
In our LLDP probe, we use the couple chassis ID/chassis ID type:

{% highlight golang %}
	chassis := p.getOrCreate(graph.GenID(chassisID, lldpLayer.ChassisID.Subtype.String()), chassisMetadata)
{% endhighlight %}

The node for the port is handled pretty much the same way.
{% highlight golang %}
	port := p.getOrCreate(graph.GenID(
		chassisID, lldpLayer.ChassisID.Subtype.String(),
		portID, lldpLayer.PortID.Subtype.String(),
	), portMetadata)
{% endhighlight %}

Once we have our two nodes created, we create ownership and L2 links between them:
{% highlight golang %}
	if !topology.HaveOwnershipLink(p.g, chassis, port) {
		topology.AddOwnershipLink(p.g, chassis, port, nil)
		topology.AddLayer2Link(p.g, chassis, port, nil)
	}

	if !topology.HaveLayer2Link(p.g, port, n) {
		topology.AddLayer2Link(p.g, port, n, nil)
	}
{% endhighlight %}

## Conclusion

The probe is really new and may contain a few bugs but it already provides
useful topology discovery. Please report any issue you may be facing with it.

I hope this post will help you writing your own post probe for Skydive :-)
Contributions are always more than welcome.