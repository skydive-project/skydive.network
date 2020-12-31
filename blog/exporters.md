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
Instructions to developers and high level architecture of the Flow Exporter can be found at [Flow Exporter Overview](https://github.com/skydive-project/skydive-flow-exporter/blob/master/README.md).
We provide below a detailed description of the various options of the Flow Exporter, and refer to its use specifically for the Security Advisor application and for reporting VPC logs.

In order to use a Flow Exporter application, you first need to have Skydive up and running.

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
For example, different transforms exist to convert flow data into formats appropriate for Security Advisor, for VPC logs, and for AWS flow logs.
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

### Description of fields in `exporter.yml` file

Refer to the default yml files for [Security Advisor](https://github.com/skydive-project/skydive-flow-exporter/blob/master/secadvisor/secadvisor.yml.default) and [VPC logs](https://github.com/skydive-project/skydive-flow-exporter/blob/master/vpclogs/vpclogs.yml.default).

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

### Security Advisor
The Security Advisor, built using the Flow Exporter pipeline, filters the flow data obtained from Skydive, performs a data transformation, and saves the information to a storage target (e.g. AWS-S3) in GZIP compressed JSON encoding format.
This flow data may then be analyzed to determine whether some security issue must be addressed.

Additional details of the parameters and output objects of the Security Advisor, as well as instructions to run the Security Advisor, can be found at [Security Advisor blog](http://skydive.network/blog/secadvisor.html).

### VPC Logs
The VPC logs application, built using the Flow Exporter pipeline, takes the flows from a specific channel (associated with a particular tenant) and transforms the flow information into a format that can be easily consumed by the user.
This data can then be used for various purposes such as accounting, anomoly detection, and security verification.
Additional details of the parameters and output objects of vpclogs can be found at [VPClogs blog](https://github.com/skydive-project/skydive.network/blob/master/blog/vpclogs.md).

## Multiple pipelines

It is possible to run multiple pipelines simultaneously.
Prepare a separate <n>.yml for each set of subnets or whatever configuration you want.
Then run a separate instance of the Security Advisor (or other pipeline) using each of the <n>.yml configuration files.
In this way it is possible to perform different filtering for different purposes or tenants.

## Troubleshooting common problems

If the log shows a `connection refused` error, verify that the proper address of the Skydive analyzer is specified under `analyzers`.

If the log shows a credentials error, verify that the credential fields (<`access_key`, `secret_key`> for minio, and <`api_key`, `iam_endpoint`> for IBM COS) are properly set in the `yml` file.

If the log shows an overflow and states that flows were discarded, check that `max_flow_array_size` is defined to some reasonable positive number - at least the number of flows you expect to capture.

Verify that host_id is `""`.
