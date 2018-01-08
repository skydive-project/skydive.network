---
title: First steps
section: 2. Traffic capture
layout: first-steps
---

<p>For this second part we will introduce the traffic capture capabilities of Skydive.</p>
<h2>First capture</h2>
<p>
  Even if Skydive comes with a Rest API and a client allowing us to create captures thanks to the command line
  we will use the WebUI as it is the easiest way to start playing with traffic capture.
  The following tiny video shows how to start a simple capture with default options on an interface.
</p>

<p>
  <video poster="" preload="" controls="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/capture-1.webm"></video>
<p>

<p>
  Once the capture is started the interface captured get a `red point` marking that the capture is active. Then you can go to the right panel to click on the
  flow refresh button to see active flows.
</p>

<p>
  To get more details about flow a simple click on it will display the interface metadata. We won't detail here all the fields as it will
  the subject of another part but `TrackingID` and `L3TrackingID` are constant ID for a flow across multiple interfaces while `UUID`
  is the ID for the flow at a specific point of capture.
</p>

<p>
  As showed in the following video there are ways to filter, to sort flows or to select specific fields that will be displayed.
  </p>

<p>
  <video poster="" preload="" controls="" autoplay="" loop="" controlslist="nodownload" src="/assets/videos/first-steps/capture-2.webm"></video>
<p>

<div style="margin-top: 40px;">
  <p style="float:left">
    <a href="/tutorials/first-steps-1.html"><i class="fa fa-chevron-left" aria-hidden="true"> 1. Getting started</i></a>
  </p>
  <p style="float:right">
    <a href="/tutorials/first-steps-3.html">3. Traffic capture, multiple interfaces <i class="fa fa-chevron-right" aria-hidden="true"></i></a>
  </p>
</div>
