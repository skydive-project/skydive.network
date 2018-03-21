---
title: First steps
section: 1. Getting started
layout: first-steps
---

<p>For this first article we will see how it is easy to start Skydive and how to explore the topology thanks to the WebUI.
<h2>Let's start it</h2>
<p>
  Skydive comes as a single binary which includes the Analyzer part, the Agent part and the client. Here we are going to use the `allinone` mode of Skydive which
  starts everything for us. There are many way to deploy Skydive but here as we going to use it for testing purpose we are going to use to easiest one which
  is simply to download the latest static binary.
</p>

{% highlight shell %}
curl -Lo skydive https://github.com/skydive-project/skydive-binaries/raw/jenkins-builds/skydive-latest && chmod +x skydive && sudo mv skydive /usr/local/bin/

SKYDIVE_ETCD_DATA_DIR=/tmp sudo -E /usr/local/bin/skydive allinone
{% endhighlight %}

<p>
  The `SKYDIVE_ETCD_DATA_DIR` variable is just there to specify where the embedded Etcd will store its data. By default Skydive will use the loopback address.
  If you want to specify another address you can use another variable.
</p>

{% highlight shell %}
SKYDIVE_ANALYZER_LISTEN=0.0.0.0:8082 SKYDIVE_ETCD_DATA_DIR=/tmp sudo -E /usr/local/bin/skydive allinone
{% endhighlight %}

<h2>The WebUI</h2>
<p>
  Skydive is now started and its WebUI is available through the port 8082. Pointing your web browser to `http://localhost:8082` you will see something
  similar to this :
</p>

<p>
  {% lightbox /assets/images/first-steps/getting-started-1.png --thumb="/assets/images/first-steps/getting-started-1.png" --data="started-1" --alt="WebUI" %}
</p>

<p>
  The following video shows basic interaction with the WebUI. Exploration of interface metadata like the `Neighbors`, the 'Routing tables', etc.
</p>

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/getting-started.webm"></video>
<p>

<div style="margin-top: 40px;">
  <p style="float:right">
    <a href="/tutorials/first-steps-2.html">2. Traffic capture <i class="fa fa-chevron-right" aria-hidden="true"></i></a>
  </p>
</div>
