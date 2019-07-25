---
title: First steps
section: 8. Skydive as Grafana datasource
layout: first-steps
---

<p>One key feature of `Skydive` is the ability to keep track of all the modifications of the topology. This is true for the Flow meaning packet capture as well. In this tutorial we will see how to leverage this with `Grafana` in order to graph metrics such as `sFlow`, `OpenvSwitch`, RTT metrics.</p>

<h2>Skydive setup</h2>

In order to have the `Skydive` history working we need to setup a storage backend. To do so please check the 
[History and Datastore](/documentation/getting-started#history-and-datastore) section.

For the purpose of this example we will use a sandbox 
[script](https://github.com/skydive-project/skydive/blob/master/scripts/simple.sh) 
that we provide in the `Skydive` repository.

This script requires to have `OpenvSwitch` up and running. It will create 2 network namespaces connected through a `OpenvSwitch` bridge.

```
./scripts/simple.sh start 192.168.0.1/24 192.168.0.2/24
```

To remove/delete the sandbox it will be :

```
./scripts/simple.sh stop
```

Taking a look at the `Skydive` WebUI we get the following topology.

<p>
  <a href="/assets/images/first-steps/grafana-1.png" data-lightbox="WebUI-1" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-1.png"/>
  </a>
</p>

<h2>Grafana</h2>


We do provide a `Grafana` `Docker` 
[image](https://hub.docker.com/r/skydive/skydive-grafana-datasource) 
with the `Skydive` datasource enabled by default. Starting `Grafana` with `Skydive` is fairly simple.

```
docker run -d --name=grafana --net=host skydive/skydive-grafana-datasource
```

<i>The default user/password is `admin/admin`.</i>

We use the host networking so that `Grafana` will have access to our local `Skydive`.

<h3>Skydive as datasource</h3>

The `Skydive` datasource has to be add as the other `Grafana` datasources. The configuration will be really simple, we just
need to enter the `Skydive` Analyzer endpoint (http://localhost:8082, by default).

<p>
  <a href="/assets/images/first-steps/grafana-2.png" data-lightbox="WebUI-2" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-2.png"/>
  </a>
</p>

<h2>First Graph, Interface Metrics</h2>

In order to have something to graph in `Grafana` we need to generate some traffic. We could make a simple ping
from one namespace to the other :

```
sudo ip netns exec vm1 ping 192.168.0.2 -s 512
```

But We would rather use the packet injector.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/grafana-1.webm"></video>
</p>

Now we have traffic we can go to `Grafana` to get it graph. The `Skydive` datasource is using the same language used for its API. It means that
you can select the metrics using the [Gremlin]() query language.

In order to graph our packets we will select on veth pair belonging the a namespace.

```
G.V().Has('Name', 'vm1-eth0')
```

In the query editor it will look like :

<p>
  <a href="/assets/images/first-steps/grafana-3.png" data-lightbox="WebUI-3" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-3.png"/>
  </a>
</p>

Here we used the `Interface` metrics but we could use the `OpenvSwitch` metrics as well, this is just a matter of select
the `Type`.

<h2>Packet capture, sFlow, RTT, etc.</h2>

So we used the `Interfaces` metrics to graph the traffic but as `Skydive` is also about capture we will graph using this feature
in the following examples.

<h3>OpenvSwitch sFlow capture</h3>

In order to get `sFlow` packet from OpenvSwitch we need to start a capture on the bridge. `Skydive` will select `sFlow` by default with default
polling rate and sampling rate. This can be adapted using the `advanced options`.

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/grafana-2.webm"></video>
</p>


The `Gremlin` expression in the query editor will be as following with the `sFlow` :

<p>
  <a href="/assets/images/first-steps/grafana-4.png" data-lightbox="WebUI-4" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-4.png"/>
  </a>
</p>

`OpenvSwitch` reports also internal information like the memory used :

<p>
  <a href="/assets/images/first-steps/grafana-5.png" data-lightbox="WebUI-5" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-5.png"/>
  </a>
</p>

<h3>Finally RTT</h3>

`Skydive` compute the `RTT` using two-way packets, like `SYN/SYN-ACK` or `ICMP ECHO/REPLY`.

In order to get the `RTT` we start a capture on the `eth0` interface of one network namespace.

Then it is just matter of selecting the `RTT` metric in the query editor.

<p>
  <a href="/assets/images/first-steps/grafana-6.png" data-lightbox="WebUI-6" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-6.png"/>
  </a>
</p>

Let's add a bit of latency....

```
sudo tc qdisc add dev vm1-eth0 root netem delay 400ms
```

Which gives...

<p>
  <a href="/assets/images/first-steps/grafana-7.png" data-lightbox="WebUI-7" data-title="Skydive WebUI">
    <img src="/assets/images/first-steps/grafana-7.png"/>
  </a>
</p>

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-7.html"><i class="fa fa-chevron-left" aria-hidden="true"> 7. Advanced capture options</i></a>
  </p>
</div>
