---
title: VPClogs application of Skydive Flow Exporter
layout: blog-post
author: Kalman Meth
date: 31/12/2019
---

## VPC logs

VPC logs information is useful for customers on a multi-tenant cloud.
Each tenant has his own VPC (Virtual Private Cloud), and may receive a report of his network activity.
Typically, a bridge and private subnet(s) are created for each VPC.
In such virtualized environments, a VNIC (virtual Network Interface Card) is usually used to interface to each VM.
There may be a firewall installed on the bridge to limit external access to the private subnets.

### Flow Exporter

The Skydive Flow Exporter filters the flow data obtained from Skydive, performs a data transformation, and saves the information to some target (e.g. an object store).
Details of the options available for the Flow Exporter can be found in [Flow Exporter blog](https://github.com/skydive-project/skydive.network/blob/master/blog/exporters.md).

The VPC logs application, built using the Flow Exporter pipeline, takes the flows from a specific channel (associated with a particular tenant) and transforms the flow information into a format that can be easily consumed by the user.
This data can then be used for various purposes such as accounting, anomoly detection, and security verification.

### Content of VPC logs objects

The VPC Logs application (`vpclogs`), built on top of the base Skydive Flow Exporter, saves flow information to an object store in JSON encoding format.
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
| number_of_flow_logs | uint32 | Number of elements in flow_logs array |
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

These fields (`collector_crn`, `attached_endpoint_type`, etc) and their defined values would then be included in each object header produced by `vpclogs`.

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
| pipeline.transform.type        | string         | set to `vpclogs` |
| pipeline.action                | list of params | action parameters |
| pipeline.action.tid1           | string         | `TID` of first network interface |
| pipeline.action.tid2           | string         | `TID` of second network interface |
| pipeline.action.reject_timeout | int            | number of seconds to wait before declaring a flow as rejected |

The `action` operation is implemented as a mangle operation in the pipeline.
The `mangle` operation receives as input a list of transformed flow structures and may perform some post-processing, as needed, for the particular application.
In the current case, the `action` mangle operation takes the list of flows received (from the 2 network interfaces),
matches up the flows that were observed on both interfaces marking them as `accepted`,
and flags those flows that are seen on only one interface as `rejected`.
The `reject_timeout` parameter is used to specify how much time to allow a flow to not be seen on one of the interfaces before declaring it as `rejected`.

