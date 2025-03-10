# InfluxData line-protocol flavor

## Overview

ClusterCockpit uses the
[InfluxData line-protocol](https://docs.influxdata.com/influxdb/v2.1/reference/syntax/line-protocol/)
for transferring messages between its components. The line-protocol is a
text-based representation of a metric/event with a value, time and describing
tags. All metrics/events have the following format (if written to `stdout`):

```txt
<measurement>,<tag set> <field set> <timestamp>
```

Where `<tag set>` and `<field set>` are comma-separated lists of `key=value`
entries. In a mind-model, think about tags as `indices` in the database for
faster lookup and the `<field set>` as values.
The timestamp is UNIX epoch time in seconds!

We are using the tag set to add metadata information and one field for the
payload.

**Remark**: In the first iteration, we only sent metrics (number values) but we
extended the specification to messages with different purposes. The below
text was changed accordingly. The update is backward-compatible, for metrics
(number values), nothing has changed.

## Message categories

There are four line-protocol message flavors:

- **Metric**: The `field` key is `value`, the `measurement` is the metric name
- **Event**: The `field` key is `event`. Events are actionable informations. The
`measurement` is set to an event class (job, slurm, status, phases, ?? ). Additional tag
`function` to indicate the purpose, similar to a REST endpoint (for the job
class this can be start_job and stop_job).
- **Log**: The `field` key is `log`. Log messages are purely informational.
  The `measurement` is set to the component identifier [ccb, ccms, ccmc, ccem,
ccnc]. Additional tag `loglevel` to set the log level (debug, info, warn,
error).
- **Control**: The `field` key is `control`, the `measurement` is set to a
control class (rapl, freq, prefetcher, topology, config). Additional tag
`method` with on of [GET,PUT].

## Messaging subjects

ClusterCockpit uses the NATS messaging network, with the option to support other
messaging frameworks in the future. To distinguish between different message
types and easily filter the types an application is interested in the following
subject hierarchy tree is used:

```txt
<cluster name>. |
                --- metrics
                |
                --- events.[job, slurm]
                |
                --- log.[ccb, ccms, ccmc, ccem, ccnc]
                |
                --- control.[get, put]
```

## Rules valid for all message categories

Each message is identifiable by the `measurement`, and the tags
`hostname`, the `type` and, if required, a `type-id`.

### Mandatory tags per message

- `hostname`
- `type`
  - `node`
  - `socket`
  - `die`
  - `memoryDomain`
  - `llc`
  - `core`
  - `hwthread`
  - `accelerator`
- `type-id` for further specifying the type like CPU socket or HW Thread identifier

Although no `type-id` is required if `type=node`, it is recommended to send `type=node,type-id=0`.

#### Optional tags depending on the message type

In some cases, optional tags are required like `filesystem`, `device` or
`version`. While you are free to do that, the ClusterCockpit components in the
stack above will recognize `stype` (= "sub type") and `stype-id`. So
`filesystem=/homes` should be better specified as
`stype=filesystem,stype-id=/homes`.

### Message types

There exist different message types in the ClusterCockpit ecosystem, all
specified using the InfluxData line-protocol.

#### Metrics

**Identification:** `value=X` field with `X` being a number

While the measurements (metric names) can be chosen freely, there is a basic set
of measurements which should be present as long as you navigate in the
ClusterCockpit ecosystem

- `flops_sp`: Single-precision floating point rate in `Flops/s`
- `flops_dp`: Double-precision floating point rate in `Flops/s`
- `flops_any`: Combined floating point rate in `Flops/s` (often `(flops_dp * 2) + flops_sp`)
- `cpu_load`: The 1m load of the system (see `/proc/loadavg`)
- `mem_used`: The amount of memory used by applications (see `/proc/meminfo`)
- `ipc`: instructions-per-cycle metric
- `mem_bw`: Main memory bandwidth (read and write) in `MByte/s`
- `cpu_power`: Power consumption of the whole CPU package
- `mem_power`: Power consumption of the memory subsystem
- `clock`: CPU clock in `MHz`
- ...

FIXME: What about the unit??
For the whole list, see [job-data schema](../../datastructures/job-data.schema.json)

Example:

```txt
flops_any,hostname=e1208,type=core,type-id=23 value=1203.3 1740027951
```

#### Events

**Identification:** Field `event="X"` with `"X"` being the payload string.
The name (measurement) of the event message indicates the event
class. The function tag specifies the purpose (similar to REST endpoints), e.g.
`start_job`, and `stop_job` for events of class job.

Example:

```txt
job,hostname=mngmt02,type=node,type-id=0,function=stop_job event={"jobId": 69, "cluster": "ccfront", "stopTime": 1738842306, "jobState": "completed"} 1740027951
```

#### Controls

**Identification:** Field `control="X"` with `"X"` being the control request. `measurement` is
set to a control class, the tag `method` is either `GET` or `PUT`.

Example:

```txt
rapl,hostname=e1208,type=socket,type-id=2,method=GET control=intel.pkg.energy_status 1740027951
```

#### Logs

**Identification:** `log="X"` field with `"X"` being the log message. The `measurement` is
set to source component id, the tag `loglevel` is one of debug, info, warn,
error.

Example:

```txt
ccb,hostname=server01,type=node,type-id=1,loglevel=info log="component: archiver cluster: alex jobId: 232383 - archiving finished" 1740027951
```
