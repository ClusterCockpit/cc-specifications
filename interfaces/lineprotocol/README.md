# Overview

ClusterCockpit uses the [InfluxData line-protocol](https://docs.influxdata.com/influxdb/v2.1/reference/syntax/line-protocol/) for transferring metrics and events between its components. The line-protocol is a text-based representation of a metric/event with a value, time and describing tags. All metrics/events have the following format (if written to `stdout`):

```
<measurement>,<tag set> <field set> <timestamp>
```

where `<tag set>` and `<field set>` are comma-separated lists of `key=value` entries. In a mind-model, think about tags as `indices` in the database for faster lookup and the `<field set>` as metric/event values. The `<measurement>` is here used as "identifier" because it does not represent a "measurement" in all cases.

# Line-protocol in the ClusterCockpit ecosystem

In ClusterCockpit we limit the flexibility of the InfluxData line-protocol slightly. The idea is to keep the format evaluatable by different components.

Each metric is identifiable by the `measurement` (= metric name), the `hostname`, the `type` and, if required, a `type-id`.



## Mandatory tags per message:
* `hostname`
* `type`
    - `node`
    - `socket`
    - `die`
    - `memoryDomain`
    - `llc`
    - `core`
    - `hwthread`
    - `accelerator`
* `type-id` for further specifying the type like CPU socket  or HW Thread identifier

Although no `type-id` is required if `type=node`, it is recommended to send `type=node,type-id=0`.

### Optional tags depending on the message:

In some cases, optional tags are required like `filesystem`, `device` or `version`. While you are free to do that, the ClusterCockpit components in the stack above will recognize `stype` (= "sub type") and `stype-id`. So `filesystem=/homes` should be better specified as `stype=filesystem,stype-id=/homes`

## Mandatory fields per measurement:
The field key is always `value`. No other field keys are evaluated by the ClusterCockpit ecosystem.

## Message types

There exist different message types in the ClusterCockpit ecosystem.

### Metrics

**Identification:** `value=X` with `X` being a number

While the measurements (metric names) can be chosen freely, there is a basic set of measurements which should be present as long as you navigate in the ClusterCockpit ecosystem

* `flops_sp`: Single-precision floating point rate in `Flops/s`
* `flops_dp`: Double-precision floating point rate in `Flops/s`
* `flops_any`: Combined floating point rate in `Flops/s` (often `(flops_dp * 2) + flops_sp`)
* `cpu_load`: The 1m load of the system (see `/proc/loadavg`)
* `mem_used`: The amount of memory used by applications (see `/proc/meminfo`)
* `ipc`: instructions-per-cycle metric
* `mem_bw`: Main memory bandwidth (read and write) in `MByte/s`
* `cpu_power`: Power consumption of the whole CPU package
* `mem_power`: Power consumption of the memory subsystem
* `clock`: CPU clock in `MHz`
* ...

For the whole list, see [job-data schema](../../datastructures/job-data.schema.json)


### Events

**Identification:** `value="X"` with `"X"` being a string

### Controls

**Identification:** `method` tag is either `GET` or `PUT`



