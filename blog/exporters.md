---
title: How to use Flow Exporter pipeline with Skydive
layout: blog-post
author: Kalman Meth
date: 31/12/2019
---

The `Flow Exporter` filters the flow data obtained from Skydive, performs a data transformation, and saves the information to some target (e.g. an object store).
This data may then be used to perform various kinds of analyses for security, accounting, or other purposes.
We use the term `pipeline` to describe the sequence of operations performed on the data: `classify`, `filter`, `transform`, `encode`, `compress`, `store`, etc.
The modeling of the exporter as a multi-phase pipeline makes it easier to reuse specific phases while implementing new exporters.
We describe below the use of the Flow Exporter in general, and its use specifically for the Security Advisor application and for reporting VPC logs.

In order to use run a Flow Exporter application, you first need to have Skydive up and running.

### Security Advisor
The Security Advisor, built using the Flow Exporter pipeline,  filters the flow data obtained from Skydive, performs a data transformation, and saves the information to a storage target (e.g. AWS-S3) in GZIP compressed JSON encoding format.
This flow data may then be analyzed to determine whether some security issue must be addressed.

Instructions to developers to deploy and run the Secruity Advisor can be found at [Secadvisor Developer's README](https://github.com/skydive-project/skydive-flow-exporter/blob/master/secadvisor/README.md).
We present below extended instructions for users to deploy the Security Advisor.

### VPC Logs
The VPC logs application, built using the Flow Exporter pipeline, takes the flows from a specific channel (associated with a particular tenant) and transforms the flow information into a format that can be easily consumed by the user.
This data can then be used for various purposes such as accounting, anomoly detection, and security verification.

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

There are additional compiling options which enable some features. For example, to build a staticaly linked version (which can run on other machines without installing prerequisites) and to support ebpf filtering, use the following build options:

```
make WITH_EBPF=true static
```

### All-in-one version Deployment

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


## How the Flow Exporter works
The basic operations of the Flow Exporter are: `classify`, `filter`, `transform`, `mangle`, `encode`, `compress`, `store`, `write`.
Additional operations exist for particular applications built on top of these basic operations.
(See the source code for these additional operations.)
The output from one operation is fed to be the input of the next operation.
Each basic operation has a built-in default implementation.
Developers may implement their own versions of some of these operations for new applications.
Configuration parameters describing how the Flow Exporter should work are specified in a yml configuration file (see below).

### Subscribe
For a particular instance of the Flow Exporter, only a subset of the flows reported by Skydive may be of interest.
The Flow Exporter registers with Skydive on a websocket interface and may specify (in the yml file) a `capture_id` to limit the reported flows to those associated with the specified `capture_id`.

### Classify and Filter
The first stage of the Flow Exporter is the `classify` operation.
The user specifies in the yml file under the `classify` configuration parameter the network masks of the subnets that make up the user's cluster.
These network masks are used to determine whether a flow is `internal`, `egress`, or `ingress`.
The user may further specify in the yml file (under the `filter` configuration parameter) on which of these types of flows to perform the subsequent operations.
Thus a user may specify that he only wants to maintain and process `egress` or `ingress` flows.
### Transform
The `transform` operation takes each Skydive flow structure provided in its input and converts it into some other format that is appropriate for the application being run.
For example, different transforms exist to convert flow data into formats appropriate for Security Advisor, for vpc logs, and for aws flow logs.
### Mangle
The `mangle` operation receives as input a list of transformed flow structures and may perform some post-processing, as needed, for the particular application.
For example, a mangler may take from the input list multiple flows with some related characteristics and combine them into a single structure, or it may enhance the output data with additional data fields.
The output is a new list of structures in the format determined by the mangler.
### Encode
The resulting list of flows is then encoded according to the needs of the application. 
The currently supported formats are `json` and `csv`.
### Compress
The user may specify whether the resulting output should be compressed.
The currently supported compression options are `gzip` and `none`.
### Store
After the flow data has been transformed, encoded, and compressed, it is stored in an object.
The user may specify whether to store each transformed flow in a separate object or to store the information about multiple flows in a single object.
### Write
The objects containing the flow information are typically stored in some object store such as `S3`.
Alternatively, the objects can be directed to standard output to be viewed on the screen. (This is convenient for debugging purposes.)
When writing objects to an object store, it may be desirable to include some object header information.
Such object information may be prefixed to the flow information by specifying a `storeheader` section in the yml file (details below).

### Create (Security Advisor) Flow Exporter Configuration File

Start from the `secadvisor.yml.default` in the `secadvisor` directory.
Different applications can be built up by specifying different parameters in the configuration file.


```
cp secadvisor.yml.default /etc/skydive/secadvisor.yml
```

The `secadvisor.yml` is used as a parameter when running secadvisor from the command line.

The default `secadvisor.yml` has the following fields:

```
# host_id is used to reference the agent, should be unique or empty
host_id: ""
# list of analyzers for running REST requests
analyzers:
  - 127.0.0.1:8082
# analyzer credentials for websocket connection
analyzer:
  auth:
    cluster:
      username:
      password:
pipeline:
  subscriber:
    url: ws://127.0.0.1:8082/ws/subscriber/flow
    capture_id:
  # classify flows into: internal, ingress, egress and other
  classify:
    # list of internal cluster address ranges
    cluster_net_masks:
      - 10.0.0.0/8
      - 172.16.0.0/12
      - 192.168.0.0/16
  # filter out flows which match a criteria
  filter:
    # exclude the following classified tags
    excluded_tags:
      - other
  # transform to secadvisor record structure
  transform:
    type: secadvisor
    secadvisor:
      exclude_started_flows: true
  mangle:
    type: logstatus
  encode:
    type: secadvisor
    json:
      pretty: true
  compress:
    type: gzip
  store:
    type: buffered
    buffered:
      filename_prefix: logs
      dirname: bucket
      max_flows_per_object: 6000
      max_seconds_per_object: 60
      max_seconds_per_stream: 86400
      max_flow_array_size: 100000
  write:
    type: s3
    s3:
      endpoint: http://127.0.0.1:9000
      access_key: user
      secret_key: password
      region: local
      # api_key: key
      # iam_endpoint: https://iam.cloud.ibm.com/identity/token
```

### Description of fields in `exporter.yml` file

#### Synopsis

| Field Name                                       | Type           | Description |
|--------------------------------------------------|----------------|-------------|
| host_id                                          | string         | "" or unique identifier |
| analyzers                                        | list of URLs   | address of Skydive analyzer for queries |
| analyzer                                         | list of params | analyzer credentials for websocket connection |
| analyzer.auth.cluster.username                   | string         | websocket credentials |
| analyzer.auth.cluster.password                   | string         | websocket credentials |
| pipeline                                         | list of params | pipeline specification |
| pipeline.subscriber.url                          | URL            | websocket address to obtain flows from Skydive |
| pipeline.subscriber.capture_id                   | string         | Skydive capture_id to obtain only relevant flows |
| pipeline.classify                                | list of params | classify options |
| pipeline.classify.cluster_net_masks              | list of cidr   | list of net_addr/net_mask to be considered internal addresses |
| pipeline.filter                                  | list of params | filter options |
| pipeline.filter.excluded_tags                    | list of tags   | specifies which flows to remove from report; possible values: `other`, `internal`, `ingress`, `egress` |
| pipeline.transform                               | list of params | specifies how to transform the data; currently defined transforms are: `secadvisor`, `awsflowlogs`, `vpclogs` |
| pipeline.transform.type                          | string         | may be `secadvisor`, `vpclogs`, `awsflowlogs`, or `none` |
| pipeline.encode                                  | list of params | encode options |
| pipeline.encode.type                             | string         | may be `json` or `csv` |
| pipeline.encode.json                             | list of params | json options |
| pipeline.encode.json.pretty                      | bool           | specifies whether json should be pretty-printed |
| pipeline.compress                                | list of params | compress options |
| pipeline.compress.type                           | string         | may be `gzip` or `none` |
| pipeline.store                                   | list of params | storage parameters |
| pipeline.store.type                              | string         | storage specification; supported types include: `buffered`, `direct` |
| pipeline.store.buffered.filename_prefix          | string         | path name prefix of objects placed in object store bucket |
| pipeline.store.buffered.dirname                  | string         | name of bucket in object store where data will be stored |
| pipeline.store.buffered.max_flows_per_object     | int            | maximum number of flows stored in a single object |
| pipeline.store.buffered.max_seconds_per_object   | int            | maximum time lapse before placing flows in a new object |
| pipeline.store.buffered.max_seconds_per_stream   | int            | maximum time lapse before collecting objects in a separate collection (stream) |
| pipeline.store.buffered.max_flow_array_size      | int            | maximum number of flows that can be processed at a time |
| pipeline.write                                   | list of params | output (object storage) parameters |
| pipeline.write.type                              | string         | storage specification; supported types include: `s3`, `stdout` |
| pipeline.write.s3                                | list of params | `s3` storage parameters |
| pipeline.write.s3.endpoint                       | URL            | address of object store |
| pipeline.write.s3.access_key                     | string         | credentials for aws-type object store |
| pipeline.write.s3.secret_key                     | string         | credentials for aws-type object store |
| pipeline.write.s3.region                         | string         | used to access aws `region` |
| pipeline.write.s3.api_key                        | string         | credentials for IBM COS-type object store |
| pipeline.write.s3.iam_endpoint                   | string         | credentials for IBM COS-type object store |

#### Additional fields for secadvisor

| Field Name                                          | Type           | Description |
|-----------------------------------------------------|----------------|-------------|
| pipeline.transform.secadvisor                       | list of params | use Security Advisor transform; currently defined parameters are: `exclude_started_flows` |
| pipeline.transform.secadvisor.exclude_started_flows | bool           | set to `true` if you want to exclude first flow info of previously started flows from the report | 
| pipeline.transform.secadvisor.extend                | list of expr   | add miscellaneous fields to flow information |

#### Additional fields for vpclogs

| Field Name                                       | Type           | Description |
|--------------------------------------------------|----------------|-------------|
| pipeline.storeheader.type                        | string         | set to `vpclogs` |
| pipeline.storeheader.vpclogs                     | list of params | user supplied parameters to be included in object headers (e.g. `vpc_crn`) |

#### Additional description of fields

Entries that begin with '#' are comments and are ignored. Be sure to include/exclude the appropriate configuration parameters.

Each instance of secadvisor (Flow Exporter pipeline) should use a distinct host_id or should use `""` (the empty host id).
If a host_id (other than `""`) is in use by one instance of the Security Advisor (pipeline), that host_id may not be used by another instance of the Security Advisor (pipeline).

##### Object Store fields

Set `endpoint` to the object store endpoint URL, e.g. `https://s3.us.cloud-object-storage.appdomain.cloud`.
Set the credentials used for authenticating to the object store service.
You can either use AWS-style HMAC keys by setting `access_key` and `secret_key`, or use IBM IAM OAuth by setting `api_key`.
AWS-style HMAC keys and their parameters are described in [Understanding and Getting Your Security Credentials](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html).
IBM IAM keys are discussed in [Managing user API keys](https://cloud.ibm.com/docs/iam?topic=iam-userapikey#userapikey).
When using IBM IAM Oauth (when setting `api_key`), the default IBM IAM authentication endpoint will be used, if none is specified.
You can customize this endpoint by setting a different `iam_endpoint` to an alternative URL string.

Set `bucket` to the name of the destination bucket where flows will be dumped.
If your object store service requires a region specification, set `region` appropriately. This setting is not necessary when using IBM COS.

##### Classify and Filter fields

The subnets specified in `cluster_net_masks` are used to determine whether a flow is `internal`, `ingress`, or `egress`.
Enter in the list all of the subnets that you consider to be inside your domain of security.

The types of `excluded_tags` that are recognized are: `internal`, `ingress`, `egress`, `other`.
The `internal` flag refers to flows that begin and end inside your domain (as defined by `cluster_net_masks`).
The `ingress` flag refers to flows whose source is ouside your domain and whose target is inside your domain.
The `egress` flag refers to flows whose source is inside your domain and whose target is outside your domain.
The `other` flag refers to flows whose source and target are both outside your domain.
For example, if you want to capture only those flows that go between your domain and outside your domain (ingress and egress), then specify `excluded_tags` to include `internal` and `other`.

##### Transform fields

The use of field `transform.secadvisor.extend` to add miscellaneous fields to the output information is discussed below under the section describing the flow output of the Security Advisor and its extension.

##### Store Buffered fields

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
AWS-style keys and their parameters are described in [Understanding and Getting Your Security Credentials](`https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html`).

### Setup IBM COS

In IBM Cloud, Create an Object Store resource. The Lite plan gives limited resources for free.
In the Object Store, create a bucket to hold the Skydive flow information. For simple testing, you can choose Single Site resiliency.
Look under Bucket Configuration to see the Public endpoint (url) where the bucket is accessed. This endpoint information needs to go into the `secadvisor.yml` `endpoint` field.
Go to the `Service Credentials` panel and create a new credential. Click on `View Credential` to get the details of the credential. The apikey needs to go into the `secadvisor.yml` `api_key` field.
In the `secadvisor.yml` file, uncomment the `iam_endpoint` field and set it to `https://iam.cloud.ibm.com/identity/token`.

### Setup Minio

For running tests on a local machine, it is possible to set up a local Minio object store.

For details, see the instructions found at [Secadvisor Developer's README](https://github.com/skydive-project/skydive/tree/master/contrib/exporters/secadvisor/README.md).

## Multiple pipelines

It is possible to run multiple pipelines simultaneously.
Prepare a separate <n>.yml for each set of subnets or whatever configuration you want.
Then run a separate instance of the Security Advisor (or other pipeline) using each of the <n>.yml configuration files.
In this way it is possible to perform different filtering for different purposes or tenants.

## Deploy the Security Advisor pipeline

Build and run the exporter in the secadvisor directory: skydive-flow-exporter/secadvisor.

```
cd skydive-flow-exporter/secadvisor
make static
```

The binary is created in the same directory: skydive-flow-exporter/secadvisor.

### Activate the Security Advisor

```
./secadvisor /etc/skydive/secadvisor.yml
```

Note that it is possible to run several instances of the Security Advisor at the same time, each one with a different configuration file.
This is useful to capture different flows for different purposes and to perform different filtering (e.g. based on specified `capture_id `, `cluster_net_masks` and `filter` parameters).

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

On another machine, run the iperf client:

```
iperf3 -c <address-of-iperf-server> -t 1000
```

It is also possible to generate network traffic using the inject feature of Skydive, either from the command line or through the Skydive GUI.
Go the `Generator` page and specify the source and destination nodes between which to generate the network traffic, as well as the characterization of the flow you want to generate.

### Observe Security Advisor objects being created

Objects are created in the Object Store only if there is flow information to be saved.

If using the default parameters, the output log of secadvisor should show that an object is being sent to the object store about once per minute.
(This can be changed by setting the configuration parameter `flow.update` in `/etc/skydive/skydive.yml`.)
Check your bucket in your object store to verify that you see a new object about once per minute.
Stop the secadvisor or stop captures in the GUI to stop the creation of objects in the object store.

## Content of Security Advisor objects

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
- B - destination port

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

UpdateCount - number of times we have already reported on this flow.

NodeType - type of Skydive endpoint such as `device`, `switch`, `tun`, `bridge`, etc.

Extend - miscellaneous fields define by the user, as described in the next section.

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

The gremlin expresion uses the [golang template feature](https://golang.org/pkg/text/template/), where template expressions are enclosed in {{ }}.
The fields surrounded by {{ }} are taken as the names of fields in the flow information provided by the `transform` (described in the previous subsection) and are replaced with their actual values, before evaluating the gremlin expression and placing the result in a new field which is added to the flow information.
For example, the gremlin expression may look like this:

```
    - AA_Name=G.V().Has('RoutingTables.Src','{{.Network.A}}').Values('Host')
    - BB_Name=G.V().Has('RoutingTables.Src','{{.Network.B}}').Values('Host')
```

The result is that the value of Network.A (in the above example: 169.45.67.210) is inserted in the gremlin expression, the gremlin expression is then evaluated (or obtained from a cache), and the resulting value (the name of the host holding the network interface) is then placed in the field `AA_Name` under the `Extend` field of the flow information.

The substitution string refers to fields that already exist in the data provided by the particular transform.
If needed, be sure to put quotes around the substitution results.
It is recommended to use only single quotes in the gremlin expression.

## VPC logs

VPC logs information is useful for customers on a multi-tenant cloud.
Each tenant has his own VPC (Virtual Private Cloud), and may receive a report of his network activity.
Typically, a bridge and private subnet(s) are created for each VPC.
In such virtualized environments, a VNIC (virtual Network Interface Card) is usually used to interface to each VM.
There may be a firewall installed on the bridge to limit external access to the private subnets.

### Content of VPC logs objects

The VPC Logs application (`vpclogs`), built on top of the base Skydive flow exporter, saves flow information to an object store in JSON encoding format.
Each object created by `vpclogs` contains the flows that were captured and filtered for a particular VPC, according to the parameters set in the `vpclogs.yml` configuration file.
An example of entries in an object created by `vpclogs` might be the following:

```
{
    "version": "0.0.1",
    "attached_endpoint_type": "vnic",
    "capture_end_time": "2008-09-15T15:53:00Z",
    "capture_start_time": "2008-09-15T15:00:00Z",
    "network_interface_id": "crn",
    "collector_crn": "crn",
    "vsi_crn": "crn",
    "vpc_crn": "crn"
    "state": "ok",
    "flow_logs": [
        {
            "start_time": "2008-09-15T15:53:00.000000Z",
            "end_time": "2008-09-15T15:40:00.000000Z",
            "connection_start_time": "2008-09-15T15:00:00Z",
            "direction": "outbound",
            "action": "accepted",
            "was_terminated": false,
            "was_initiated": true,
            "ip_protocol": 4,
            "initiator_ip": "1.2.3.4",
            "initiator_port": 20033,
            "target_ip": "5.6.7.8",
            "target_port": 80,
            "transport_protocol": 6,
            "bytes_from_initiator": 12000,
            "packets_from_initiator": 2212,
            "bytes_from_target": 323232,
            "packets_from_target": 3232
            "cumulative_packets_from_initiator": 2212,
            "cumulative_packets_from_target": 3232,
            "cumulative_bytes_from_target": 323232,
            "cumulative_bytes_from_initiator": 12000,
        }
    ],
}
```

There is an object header with general information about the flows contained therein, and there is a list of flow logs information.

### Per-flow parameters

| Field Name                        | Type   | Description |
|-----------------------------------|--------|-------------|
| start_time                        | string | When first byte in flow-log was captured / seen in datapath (RFC 3339 Date and Time (UTC)) |
| end_time                          | string | When last byte in flow-log was captured / seen in datapath (RFC 3339 Date and Time (UTC)) |
| connection_start_time             | string | When first byte in flow-log’s connection was captured / seen in datapath (RFC 3339 Date and Time (UTC)) |
| direction                         | string | "inbound" or "outbound" - If first packet on connection was received by the VNIC, direction is "inbound". If first packet was sent by the VNIC, direction is "outbound" |
| action                            | string | "accepted" or "rejected" - Whether the traffic summarized by this flow-log was Accepted or Rejected |
| initiator_ip                      | string | (IPv4 address) Source-IP as it appears in first packet processed by VNIC on this connection. If Direction=="outbound", this is a private IP associated with the VNIC |
| target_ip                         | string | (IPv4 address) Dest-IP as it appears in first packet processed by VNIC on this connection. If Direction=="inbound", this is a private IP associated with the VNIC |
| initiator_port                    | uint16 | TCP/UDP source-port as it appears in first packet processed by this VNIC on this connection |
| target_port                       | uint16 | TCP/UDP dest-port as it appears in first packet processed by this VNIC on this connection |
| transport_protocol                | uint8  | IANA protocol number (TCP or UDP) |
| ether_type                        | string | (“IPv4”) - In the future may be extended to IPv6 |
| was_initiated                     | bool   | Was the connection initiated in this flow-log: SYN encountered or actual connection established? |
| was_terminated                    | bool   | Was the connection terminated (E.g. timeout/RST/Final-FIN) |
| bytes_from_initiator              | uint64 | Count of bytes on connection in this flow-log’s time-window, in the direction Initiator to Target |
| packets_from_initiator            | uint64 | Count of packets on connection in this flow-log’s time-window, in the direction Initiator to Target |
| bytes_from_target                 | uint64 | Count of bytes on connection in this flow-log’s time-window, in the direction Target to Initiator |
| packets_from_target               | uint64 | Count of packets on connection in this flow-log’s time-window, in the direction Target to Initiator |
| cumulative_bytes_from_initiator   | uint64 | Count of bytes since connection initiated, in direction Initiator to Target |
| cumulative_packets_from_initiator | uint64 | Count of packets since connection initiated, in direction Initiator to Target |
| cumulative_bytes_from_target      | uint64 | Count of bytes since connection initiated, in direction Target to Initiator |
| cumulative_packets_from_target    | uint64 | Count of packets since connection initiated, in direction Target to Initiator |

### Object Header parameters

| Field Name          | Type   | Description |
|---------------------|--------|-------------|
| version             | string | major.minor.patch |
| capture_start_time  | string | RFC 3339 Date and Time (UTC) |
| capture_end_time    | string | RFC 3339 Date and Time (UTC) |
| state               | string | “ok” or “skip data” there may be flow-log data missing (both since last object AND in this object) |
| number_of_flow_logs | uint32 | Number of elements in flowLogs array |
| flow_logs           | array of JSON objects  | May be empty array – indicates ‘no traffic’ |

Additional fields (such as VPC global information) may be defined in the `vpclogs.yml` file to be included in the object header.
The syntax in the `vpclogs.yml` file to add these additional fields in specified as follows.

| Field Name                   | Type           | Description |
|------------------------------|----------------|-------------|
| pipeline.storeheader         | list of params | Add header to flows storage object |
| pipeline.storeheader.type    | string         | may be vpclogs or none |
| pipeline.storeheader.vpclogs | list of params | user specified fields to be included in object header |

The corresponding entries in the `vpclogs.yml` file would then look like something like this:

```
pipeline:
  transform:
    type: vpclogs
  storeheader:
    type: vpclogs
    vpclogs:
      collector_crn: xxxxxx
      attached_endpoint_type: xxxxxx
      network_interface_id: xxxxxx
      instance_crn: xxxxxx
      vpc_crn: xxxxxx
```

### Action parameter

When running in a VPC, it may be desirable to know which flows were allowed through the firewall and which flows were blocked.
This information is captured by the `action` parameter in the `vpclogs` data, which may be `accepted` or `rejected`.
In some environments, the `action` information may be directly available from the firewall.
In a generic cloud, where `action` information is not available from the firewall, the flows exporter provides a component fo compute this field. 
The setup requires that the flows be captured at two network interfaces, one on either side of the firewall.
Flows that appear at both interfaces are deemed to be `accepted`.
Flows that appear at one interface but not at the other interface are deemed to be `rejected`.
In order to make this work, the flows should be captured at only these two interfaces.
Each of these network interfaces has a Skydive-defined `TID` (Topology ID).
To specify the capture of flows at the two specific interfaces, use the following command.

```
skydive client capture create --extra-tcp-metric --gremlin "G.V().Has('TID', Within('<tid1>', '<tid2>'))"
```

In the above command, `<tid1>` and `<tid2>` are the Skydive TIDs of the network interfaces on the two sides of the firewall.
This command produces a `<capture_id>`.

In the vpclogs.yml file, add the additional fields:

```
pipeline:
  subscriber:
    url: ws://127.0.0.1:8082/ws/subscriber/flow
    capture_id: <capture_id>
  transform:
    type: vpclogs
  mangle:
    type: action
  action:
    tid1: <tid1>
    tid2: <tid2>
    reject_timeout: 5
```

where `<tid1>`, `<tid2>`, and `<capture_id>` have been replaced with the proper values.

| Field Name                     | Type           | Description |
|--------------------------------|----------------|-------------|
| pipeline.subscriber.capture_id | string         | Skydive capture_id to obtain only relevant flows |
| pipeline.transform.type        | string         | seet to `vpclogs` |
| pipeline.action                | list of params | action parameters |
| pipeline.action.tid1           | string         | `TID` of first network interface |
| pipeline.action.tid2           | string         | `TID` of second network interface |
| pipeline.action.reject_timeout | int            | number of seconds to wait before declaring a flow as rejected |

The `action` operation is implemented as a mangle operation in the pipeline.
The `mangle` operation receives as input a list of transformed flow structures and may perform some post-processing, as needed, for the particular application.
In the current case, the `action` mangle operation takes the list of flows received (from the 2 network interfaces),
matches up the flows that were observed on both interfaces marking them as `accepted`,
and flags those flows that are seen on only one interface as `rejected`.
The `reject_timeout` parameter is used to specify how much time to allow a flow to be seen on both interfaces before declaring it as `rejected`.

## Troubleshooting common problems

If the log shows a `connection refused` error, verify that the proper address of the Skydive analyzer is specified under `analyzers`.

If the log shows a credentials error, verify that the credential fields (<`access_key`, `secret_key`> for minio, and <`api_key`, `iam_endpoint`> for IBM COS) are properly set in the `secadvisor.yml` file.

If the log shows an overflow and states that flows were discarded, check that `max_flow_array_size` is defined to some reasonable positive number - at least the number of flows you expect to capture.

Verify that host_id is `""`.
