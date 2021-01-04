---
title: How to use Security Advisor with Skydive
layout: blog-post
author: Kalman Meth
date: 31/07/2019
---


The Security Advisor filters the flow data obtained from Skydive, performs a data transformation, and saves the information to an object store in GZIP compressed JSON encoding format.
This data may then be used to perform various kinds of analyses for security, accounting, or other purposes.
The Security Advisor is built on top of the Skydive Flow Exporter.
Instructions to developers and high level architecture of the Flow Exporter can be found at [Flow Exporter Overview](https://github.com/skydive-project/skydive-flow-exporter/blob/master/README.md).
We present here extended instructions for users to deploy Skydive and the Security Advisor.

We use the term `pipeline` to describe the sequence of operations performed on the data: classify, filter, transform, encode, store.

## Deploy Skydive

You can either download a statically linked version of Skydive or you can build it on your own.

To download a pre-compiled statically linked version of Skydive, run the following command.

```
curl -Lo skydive https://github.com/skydive-project/skydive-binaries/raw/jenkins-builds/skydive-latest && chmod +x skydive && sudo mv skydive /usr/local/skydive/
```

To build your own version of Skydive, follow the instructions on [Build Documentation](/documentation/build) to install prerequisites and prepare your machine to build the Skydive code.
Enter the Skydive root directory and build the code.

```
cd $GOPATH/src/github.com/skydive-project/skydive
make
cp etc/skydive.yml.default /etc/skydive/skydive.yml
# adjust settings in skydive.yml, if desired
```

The skydive binary is created in $GOPATH/bin/skydive.

### All-in-one version

```
sudo skydive allineone -c /etc/skydive/skydive.yml
```

### Multi-node Deployment

Alternatively, run a Skydive analyzer on one host and a Skydive agent on each of your hosts.

On one machine:

```
sudo skydive analyzer -c /etc/skydive/skydive.yml
```
On each machine:

```
sudo skydive agent -c /etc/skydive/skydive.yml
```

Be sure to set the field `analyzers` in the `skydive.yml` to point to the analyzer.


## Create Security Advisor (Pipeline) Configuration File

Start from the `secadvisor.yml.default` in the secadvisor directory.


```
cp secadvisor.yml.default /etc/skydive/secadvisor.yml
```

The `secadvisor.yml` is used as a parameter when running secadvisor from the command line.

The default `secadvisor.yml` has the following fields:

```
host_id: ""
analyzers:
  127.0.0.1:8082
pipeline:
  analyzer:
    subscriber_url: ws://127.0.0.1:8082/ws/subscriber/flow
    subscriber_username:
    subscriber_password:
  classify:
    # cluster_net_masks:
      # - 10.0.0.0/8
      # - 172.16.0.0/12
      # - 192.168.0.0/16
  filter:
    # excluded_tags:
      # - internal
      # - other
      # - ingress
      # - egress
  transform:
    sa:
      # exclude_started_flows: true
  store:
    type: s3
    s3:
      # -- client parames --
      endpoint: http://127.0.0.1:9000
      region: local
      bucket: bucket
      access_key: user
      secret_key: password
      # api_key: key
      # iam_endpoint: https://iam.cloud.ibm.com/identity/token
      object_prefix: logs
      # -- bulk store params --
      max_flows_per_object: 6000
      max_seconds_per_object: 60
      max_seconds_per_stream: 86400
      max_flow_array_size: 100000
```

### Description of fields in `secadvisor.yml` file

Entries that begin with '#' are comments and are ignored. Be sure to include/exclude the appropriate configuration parameters.

Each instance of secadvisor should use a distinct host_id or should use `""` (the empty host id).
If a host_id (other than `""`) is in use by one instance of the Security Advisor (pipeline), that host_id may not be used by another instance of the Security Advisor (pipeline).

Set `endpoint` to the object store endpoint URL, e.g. `https://s3.us.cloud-object-storage.appdomain.cloud`.
Set the credentials used for authenticating to the object store service.
You can either use AWS-style HMAC keys by setting `access_key` and `secret_key`, or use IBM IAM OAuth by setting `api_key`.
When using IBM IAM Oauth (when setting `api_key`), the default IBM IAM authentication endpoint will be used: `https://iam.ng.bluemix.net/oidc/token`, if none is specified.
You can customize this endpoint by setting a different `iam_endpoint` to an alternative URL string.

Set `bucket` to the name of the destination bucket where flows will be dumped.
If your object store service requires a region specification, set `region` appropriately. This setting is not necessary when using IBM COS.

The subnets specified in `cluster_net_masks` are used to determine whether a flow is internal, ingress, or egress.
Enter in the list all of the subnets that you consider to be inside your domain of security.

The types of `excluded_tags` that are recognized are: internal, ingress, egress, other.
The `internal` flag refers to flows that begin and end inside your domain (as defined by `cluster_net_masks`).
The `ingress` flag refers to flows whose source is ouside your domain and whose target is inside your domain.
The `egress` flag refers to flows whose source is inside your domain and whose target is outside your domain.
The `other` flag refers to flows whose source and target are both outside your domain.
For example, if you want to capture only those flows that go between your domain and outside your domain (ingress and egress), then specify `excluded_tags` to include `internal` and `other`.

Be sure to set the parameters `max_flow_array_size`, etc, to reasonable values.

The `max_flow_array_size` specifies the maximum number of flows that will be stored in each iteration.
If `max_flow_array_size` is set to 0 (or not set at all), then the number of flows that can be stored is 0, and hence no useful information will be saved in the object store.
This will result in an error.

The `max_flows_per_object` parameter specifies the number of flows that may stored in a single object. If there are more flows found in a single iteration, then several objects will be created, each with up to `max_flows_per_object` flows in them. In order to have all the flows stored in a single object, be sure that this parameter is set sufficiently large.
If `max_flows_per_object` is set to 0 (or not set at all), then each object will be able to hold 0 flows; i.e. no flows will be able to be stored in any object.
This will result in an error.

The flow information is collected in groups of objects (called streams) determined by the `max_seconds_per_stream` parameter.
All the objects generated within the number of seconds specified in the `max_seconds_per_stream` parameter are collected under the same heading (stream) in the object url path.
If `max_seconds_per_stream` is set to a very large number, then all of the flows captured in the current run of the Security Advisor will be included in the same collection (stream)
Furthermore, a single object may contain multiple statistics of the flows - all those collected within `max_seconds_per_object` seconds. Thus if `max_seconds_per_object` is set to 200 and the flows statistics  are captured once a minute, then each object will contain 3 or 4 sets of statistics - whatever was collected within 200 seconds. 

## Setup Object Store

The Security Advisor saves data to an S3-type object store.
The parameters to access the object store must be provided in the `secadvisor.yml` configuration file.
In case the user does not already have an object store, we show below how to create an object store for testing purposes.

You can either use AWS-style HMAC keys by setting `access_key` and `secret_key`, or use IBM IAM OAuth by setting `api_key`.

### Setup IBM COS

In IBM Cloud, Create an Object Store resource.
In the Object Store, create a bucket to hold the Skydive flow information. For simple testing, you can choose Single Site resiliency.
Look under Bucket Configuration to see the Public endpoint (url) where the bucket is accessed. This endpoint information needs to go into the `secadvisor.yml` `endpoint` field.
Go to the `Service Credentials` panel and create a new credential. Click on `View Credential` to get the details of the credential. The apikey needs to go into the `secadvisor.yml` `api_key` field.
In the `secadvisor.yml` file, uncomment the `iam_endpoint` field and set it to `https://iam.cloud.ibm.com/identity/token`.

### Setup Minio

For running tests on a local machine, it is possible to set up a local Minio object store.

For details, see the instructions in [Flow Exporter Overview](https://github.com/skydive-project/skydive-flow-exporter/blob/master/README.md).

## Deploy the Security Advisor pipeline

Build and run the pipeline in the secadvisor directory: skydive-flow-exporter/secadvisor

```
cd skydive-flow-exporter/secadvisor
make static
```

The binary is created in the directory: go/bin

### Activate the Security Advisor

```
secadvisor /etc/skydive/secadvisor.yml
```

Note that it is possible to run several instances of the Security Advisor at the same time, each one with a different configuration file.
This is useful to capture different flows for different purposes and to perform different filtering (e.g. based on specified `cluster_net_masks` and `filter` parameters).

## Generate and Capture Flows

### Capture Flows

Via the Skydive WebUI setup captures and generate traffic which should
result in the secadvisor pipeline sending flows to the ObjectStore.

To connect to the GUI, open a web browser to the address of the Skydive analyzer at port 8082. If running the browser on the same machine as the analyzer, then connect to localhost:8082.
You should see the topology of your network, perhaps something like the following image.

<p class="center">
  <a href="/assets/images/blog/capture-future-1.png" data-lightbox="Skydive Topology" data-title="Skydive topology view">
    <img src="/assets/images/blog/capture-future-1.png"/>
  </a>
</p>


Make sure you are on the `Captures` view and press on `Create`.

Fill in the `Targets` fields by placing the cursor in the first `Interface` field and then use the mouse to point to your network endpoint that you want to capture.
To capture a particular flow end-to-end, fill in the second `Interface` field in a similar manner with the other network endpoint.
Then press the `Start` button on the GUI.

Alternatively, you can start the capture of flows from the command line with a commond like the following:

```
skydive client capture create --gremlin "G.V().Has('Type', 'device', 'Name', 'eth0')" --type pcap 
```

The command specifies to capture all flows that match the gremlin expression - in this case, all devices that have name 'eth0'.

### Generate Flows

Generate some network traffic on the interfaces you specified to capture.
For example, run some iperf traffic between entities and capture the flow via the Skydive GUI.

To install `iperf3` (on ubuntu/debian):

```
sudo apt-get install iperf3
```

Run iperf server:

```
iperf3 -s
```

Either on the same machine or on another machine, run the iperf client:

```
iperf3 -c <address-of-iperfserever> -t 1000
```

It is also possible to generate network traffic using the inject feature of Skydive, either from the command line or through the Skydive GUI.
Go the `Generator` page and specify the source and destination nodes between which to generate the network traffic, as well as the characterization of the flow you want to generate.

### Observe Security Advisor objects being created

Objects are created in the Object Store only if there is flow information to be saved.

The output log of secadvisor should show that an object is being sent to the object store about once per minute.
Check your bucket in your object store to verify that you see a new object about once per minute.
Stop the secadvisor or stop captures in the GUI to stop the creation of objects in the object store.

### Content of Security Advisor objects

The Security Advisor saves flow information to an object store in GZIP compressed JSON encoding format.
Each object created by the Security Advisor contains the flows that were captured and filtered according to the parameters set in the `secadvisor.yml` configuration file.
An example of entries in an unzipped object created by Security Advisor might be the following:

```
{
	"UUID": "7521e85422cbc042",
	"LayersPath": "Ethernet/IPv4/TCP",
	"Version": "1.0.8",
	"Status": "UPDATED",
	"Network": {
		"Protocol": "IPV4",
		"A": "169.45.67.210",
		"B": "169.44.184.135",
	},
	"Transport": {
		"Protocol": "TCP",
		"A": "32396",
		"B": "4723"
	},
	"LastUpdateMetric": {
		"ABPackets": 0,
		"ABBytes": 112,
		"BAPackets": 0,
		"BABytes": 178,
		"Start": 1551199625889,
		"Last": 1551199655889
	},
	"Metric": {
		"ABPackets": 2,
		"ABBytes": 178,
		"BAPackets": 3,
		"BABytes": 244,
		"Start": 0,
		"Last": 0
	},
	"Start": 1551199609804,
	"Last": 1551199631044,
	"updateCount": 1,
	"NodeType": "device"
}
```

For each flow we have the following fields and types, all collected in JSON encoding format.

```
        UUID             string
        LayersPath       string
        Version          string
        Status           string
        FinishType       string
        Network          *SecurityAdvisorFlowLayer
        Transport        *SecurityAdvisorFlowLayer
        LastUpdateMetric *flow.FlowMetric
        Metric           *flow.FlowMetric
        Start            int64
        Last             int64
        UpdateCount      int64
        NodeType         string
        Extend           map[string]interface{}
```

UUID - unique ID of the flow.

LayersPath - represents the layers of the network stack; typical values might be `Ethernet/IPv4/TCP`, `Ethernet/IPv4/UDP/DNS`, `Ethernet/IPv4/ICMPv4`.

Version - version of the Security Advisor pipeline being run.

Status - status of the flow; may be `STARTED`, `ENDED`, or `UPDATED`.

FinishType - indicates how a flow ended; may be `SYN_FIN`, `SYN_RST`, `Timeout`, `OVERFLOW`, or empty if flow still on-going.

Network - provides network layer information; contains the following fields:
- Protocol - e.g. IPV4 or IPV6
- A - source address
- B - destination address

Transport - provides transport layer information; contains the following fields:
- Protocol - e.g. TCP or UDP
- A - source port
- B destination port

LastUpdateMetric - represents delta relative information only for this latest update from source (`A`) to destination (`B`).
It contains the following fields pertaining to the most recent update interval:
- ABPackets - number of data packets sent from `A` to `B`
- ABBytes - number of bytes sent from `A` to `B`
- BAPackets - number of data packets sent from `B` to `A`
- BABytes - number of bytes sent from `B` to `A`
- Start - start time of this measurement interval
- Last - end time of this measurement interval

Metric - represents cumulative information for this flow from source (`A`) to destination (`B`).

Start - time when this flow was first detected.

Last - time when last packet for this flow was detected.

UpdateCount - number of updates in this report.

NodeType - type of Skydive endpoint such as `device`, `switch`, `tun`, `bridge`, etc.

Extend - miscellaneous fields defined by the user, as described in the next section.

### Adding miscellaneous fields to Security Advisor output via configuation

We added to the secadvisor pipeline the capability to extend the data output by specifying gremlin expressions with substitution in the yml file to generate additional fields in the output flow information.
The basic syntax in the yml file looks like the following:

```
transform:
  type: secadvisor
  secadvisor:
    exclude_started_flows: false
    extend:
      - VAR_NAME1=<gremlin expression with substitution strings>
      - VAR_NAME2=<gremlin expression with substitution strings>
```

The gremlin expresion uses the [golang template feature](https://golang.org/pkg/text/template/), where template expressions are enclosed in \{\{ \}\}.
The fields surrounded by \{\{ \}\} are taken as the names of fields in the flow information provided by the `transform` (described in the previous subsection) and are replaced with their actual values, before evaluating the gremlin expression and placing the result in a new field which is added to the flow information.
For example, the gremlin expression may look like this:

```
    - AA_Name=G.V().Has('RoutingTables.Src','\{\{.Network.A\}\}').Values('Host')
    - BB_Name=G.V().Has('RoutingTables.Src','\{\{.Network.B\}\}').Values('Host')
```

The result is that the value of Network.A (in the above example: 169.45.67.210) is inserted in the gremlin expression, the gremlin expression is then evaluated (or obtained from a cache), and the resulting value (the name of the host holding the network interface) is then placed in the field `AA_Name` under the `Extend` field of the flow information.

The substitution string refers to fields that already exist in the data provided by the particular transform.
If needed, be sure to put quotes around the substitution results.
It is recommended to use only single quotes in the gremlin expression.


## Multiple pipelines

It is possible to run multiple pipelines simultaneously. Prepare a separate <n>.yml for each set of subnets or whatever configuration you want. Then run a separate instance of the Security Advisor (or other pipeline) using each of the <n>.yml configuration files. In this way it is possible to perform different filtering for different purposes or tenants.

## Troubleshooting common problems

If the secadvisor log shows a `connection refused` error, verify that the proper address of the Skydive analyzer is specified under `analyzers`.

If the secadvisor log shows a credentials error, verify that the credential fields (<`access_key`, `secret_key`> for minio, and <`api_key`, `iam_endpoint`> for IBM COS) are properly set in the `secadvisor.yml` file.

If the secadvisor log shows an overflow and states that flows were discarded, check that `max_flow_array_size` is defined to some reasonable positive number - at least the number of flows you expect to capture.

Verify that host_id is `""`.
