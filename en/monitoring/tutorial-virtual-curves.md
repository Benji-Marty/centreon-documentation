---
id: create-virtual-curves
title: Extend your Centreon metrics with Virtual Curves
---

In the era of observability, a lot of metrics are collected on a wide range of devices. Sometimes, metrics gathered from a Centreon Host are even more interesting when compared to or associated with others.

This How-to will guide you through the creation of a single check aggregating metrics from multiple Hosts and Services. 

## Typical use cases 

Use cases below are some examples of what we've done so far with our community users and customers:

 * Sum different network devices bandwidth utilization (example: incoming or outgoing traffic of a specific network zone).
 * Sum storage utilization (Space used or provisioned across all your VMWare datastores).
 * Get average, min, max of individual Services' metrics (example: connected users or license usage).
 * Perform specific mathematical operations on metrics to create new ones (example: Convert power usage into CO2 emanation).

## Concepts

That screenshot of a Service detailed page highlights the main Centreon concepts you must get to follow this guide. Pretty simple right?  

![image](../../assets/monitoring/virtualcurves//virtual-curves-01_concepts.png)

## How it works 

The Plugin connects to the Centreon database and reads a configuration file to perform advanced computing on metrics. 

It uses the database to fetch metrics' current values during its execution.

For its parts, the configuration file defines:

 * Which Hosts, Services, Metrics are important? 
 * How to mold raw metrics?
 * How to output the result?

## Configuration file

The case here is to get the total IN and OUT bandwidth usage on these two hosts: 

![image](../../assets/monitoring/virtualcurves/virtual-curves-02_target_resources.png)

Here is how the configuration file looks like: 

```json
{
        "selection": {
                "traffic_in_node_1": {
                        "host_name": "HQ-PXY-Inet-1",
                        "service_name": "Traffic-data",
                        "metric_name": "traffic_in",
                        "display": false
                },
                "traffic_in_node_2": {
                        "host_name": "HQ-PXY-Inet-2",
                        "service_name": "Traffic-data",
                        "metric_name": "traffic_in",
                        "display": false
                },
                "traffic_out_node_1": {
                        "host_name": "HQ-PXY-Inet-1",
                        "service_name": "Traffic-data",
                        "metric_name": "traffic_out",
                        "display": false
                },
                "traffic_in_node_2": {
                        "host_name": "HQ-PXY-Inet-2",
                        "service_name": "Traffic-data",
                        "metric_name": "traffic_out",
                        "display": false
                }
        },
        "virtualcurve": {
                "sum_traffic_in": {
                        "pattern": "^traffic_in$",
                        "aggregation":"sum",
                        "formatting": {
                                "change_bytes": true,
                                "printf_msg": "Incoming traffic SUM is: %s %s/s",
                                "printf_var": "$self->{result_values}->{value}, $self->{result_values}->{unit}"
                        }
                },
                "sum_traffic_out": {
                        "pattern": "^traffic_out$",
                        "aggregation":"sum",
                        "formatting": {
                                "change_bytes": true,
                                "printf_msg": "Outgoing traffic SUM is: %s %s/s",
                                "printf_var": "$self->{result_values}->{value}, $self->{result_values}->{unit}"
                        }
                }
        }
}
```

### Basic use

The "selection" JSON node is meant to describe metrics to be selected: 

 * "traffic_node_1" & "traffic_node 2" are selection identifiers. Both have their keys:
    * "host_name": The Hostname in Centreon.
    * "service_name": The Service name in Centreon.
    * "metric_name": The metric name (you can use a regex filter).
    * "display": Define if this this metric is visible.

The formatting sub-section defines how I want to display the message:
 * "printf_msg": Printf expression.
 * "printf_var": Scalar to use within Printf substitution. Stick to the example as they are the only ones available at the moment.
 * "change_bytes": Does the value printed in a human-readable format. Possible values are: true, false. 

The "virtualcurves" JSON node is meant to describe the new curves to be created: 
 * "sum_traffic_in" & "sum_traffic_out" are virtual curves identifiers. Both have their keys:
    * "pattern": Pattern to pick a part of previously selected metrics inside the virtual curve. 
    * "aggregation": Aggregation you want to apply. Possible values are: 'min', 'max,' 'sum', 'avg' or 'none'.

The formatting sub-section defines how I want to display the message:
 * "printf_msg": Printf expression.
 * "printf_var": Scalar to use within Printf substitution. Stick to the example as they are the only ones available at the moment.
 * "change_bytes": Does the value printed in a human-readable format. Possible values are: true, false. 

### Advanced use

The "selection" JSON node may be substituted by "filters". It allows SQL '%' wildcard utilization and makes the configuration much simpler. 

```json
        "filters": {
                "host": "HQ-PXY-Inet%",
                "service": "Traffic-data",
                "metric": "traffic_%",
                "display": false
        },
```

Note variations in the name of the keys:
 * "host": Hostname filter
 * "service": Service filter
 * "metric": Metric filter

An optional "formatting" JSON node permits setting display globally. It is most of the time used to print a user-friendly message. 

```json
        "formatting": {
                "custom_message_global": "Total traffic bandwidth is OK"
        }
```

## Setup and practice 

### Prerequisites

The configuration file must be readable by the centreon-engine user. 

The Poller executing the check must be able to connect to the centreon_storage database over 3306/TCP port with values supplied through --username and --password options.

The SQL user must hold required privileges to "SELECT" data within index_data and metrics tables part of the centreon_storage database.

### Installation 

Create a directory to store your config files:

```bash
mkdir /opt/virtual_service_config/
```

Set the centreon-engine user as owner:

```bash
chown centreon-engine. /opt/virtual_service_config/
```


Install the Plugin-Pack: 

<!--DOCUSAURUS_CODE_TABS-->

<!--Online IMP Licence & IT-100 Editions-->

1. Install the Plugin code on the Central or a Poller

```bash
yum install centreon-plugin-Applications-Monitoring-Centreon-SQL-Metrics
```

2. Install the 'Monitoring-Centreon-SQL-Metrics" Plugin-Pack through "Configuration > Plugin packs > Manager" page:


<!--Offline IMP License-->

1. Install the Plugin code on the Central or a Poller

```bash
yum install centreon-plugin-Applications-Monitoring-Centreon-SQL-Metrics
```

2. Install the Plugin-Pack RPM:

```bash
yum install centreon-pack-applications-monitoring-centreon-sql-metrics
```

3. Install the 'Monitoring-Centreon-SQL-Metrics" Plugin-Pack through "Configuration > Plugin packs > Manager" page:

<!--END_DOCUSAURUS_CODE_TABS-->


### Usage

### Plugin execution

You can execute the plugin like this:

```shell
/usr/lib/centreon/plugins/centreon_centreon_sql_metrics.pl \ 
--plugin=database::mysql::plugin \
--dyn-mode=apps::centreon::sql::mode::virtualservice \
--host='127.0.0.1' \
--username='centreon-read' \
--password='CentreonReadPasswd' \
--config-file='/opt/virtual_service_config/config-traffic-proxy-filters.json' \
--verbose
```

Here is the result of previous execution: 

```shell
OK: Total traffic bandwidth is OK | 'sum_traffic_in'=290765421;;;; 'sum_traffic_out'=291488113;;;;
Incoming traffic sum is: 277.30 MB/s
Outgoing traffic sum is: 277.98 MB/s
```

Add the ```--help``` flag to see all available options. 

It's possible to set WARNING and CRITICAL threshold with ```--warning/critical-metric``` or ```--warning/critical-global``` options.


@TODOinsert-screen-graph-result-here


### How does the config file modify plugin behavior?

Let's modify the configuration file and see what it changes.

* Set display as true in your filters section. Note that all selected metrics are now visible in the verbose output:  

Your "filters" section looks like this: 

```json
        "filters": {
                "host": "HQ-PXY-Inet%",
                "service": "Traffic-data",
                "metric": "traffic_%",
                "display": true
        },
```

Plugin execution now produces the output shown below: 

```shell
OK: Total traffic bandwidth is OK - All metrics are OK | 'sum_traffic_in'=290772374;;;; 'sum_traffic_out'=291485566;;;; 'HQ-PXY-Inet-1#Traffic-data#traffic_in'=289862000b/s;;;0;1000000000 'HQ-PXY-Inet-1#Traffic-data#traffic_out'=291194000b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_in'=910374b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_out'=291566b/s;;;0;1000000000
Incoming traffic sum is: 277.30 MB/s
Outgoing traffic sum is: 277.98 MB/s
Metric 'HQ-PXY-Inet-1#Traffic-data#traffic_in' value is '289862000'
Metric 'HQ-PXY-Inet-1#Traffic-data#traffic_out' value is '291194000'
Metric 'HQ-PXY-Inet-2#Traffic-data#traffic_in' value is '910374'
Metric 'HQ-PXY-Inet-2#Traffic-data#traffic_out' value is '291566'
```

* Add a global change_bytes directive and set its value to true to have a human readable output. Note the modification at the end of the verbose output. 

Your formatting section looks like this:

```json
        "formatting": {
                "custom_message_global": "Total traffic bandwidth is OK",
                "change_bytes": true
        }
```

Plugin execution now procudes the output shown below: 

```shell
OK: Total traffic bandwidth is OK - All metrics are OK | 'sum_traffic_in'=290787180;;;; 'sum_traffic_out'=291500817;;;; 'HQ-PXY-Inet-1#Traffic-data#traffic_in'=289862000b/s;;;0;1000000000 'HQ-PXY-Inet-1#Traffic-data#traffic_out'=291194000b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_in'=925180b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_out'=306817b/s;;;0;1000000000
Incoming traffic sum is: 277.32 MB/s
Outgoing traffic sum is: 278.00 MB/s
Metric 'HQ-PXY-Inet-1#Traffic-data#traffic_in' value is '276.43'
Metric 'HQ-PXY-Inet-1#Traffic-data#traffic_out' value is '277.70'
Metric 'HQ-PXY-Inet-2#Traffic-data#traffic_in' value is '903.50'
Metric 'HQ-PXY-Inet-2#Traffic-data#traffic_out' value is '299.63'
```

* Now tune the metric message to be able to display the converted unit: 

Modify the "filters" section to add a formatting sub-section: 

```json
        "filters": {
                "host": "HQ-PXY-Inet%",
                "service": "Traffic-data",
                "metric": "traffic_%",
                "display": true,
                "formatting": {
                        "printf_msg": "'%s' metric value is: %s %s/s",
                        "printf_var": "$self->{result_values}->{instance}, $self->{result_values}->{value}, $self->{result_values}->{unit}"
                }
        },
```

Plugin execution now procudes the output shown below: 

```shell
OK: Total traffic bandwidth is OK - All metrics are OK | 'sum_traffic_in'=290940974;;;; 'sum_traffic_out'=291624380;;;; 'HQ-PXY-Inet-1#Traffic-data#traffic_in'=290013000b/s;;;0;1000000000 'HQ-PXY-Inet-1#Traffic-data#traffic_out'=291316000b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_in'=927974b/s;;;0;1000000000 'HQ-PXY-Inet-2#Traffic-data#traffic_out'=308380b/s;;;0;1000000000
Incoming traffic sum is: 277.46 MB/s
Outgoing traffic sum is: 278.11 MB/s
'HQ-PXY-Inet-1#Traffic-data#traffic_in' metric value is: 276.58 MB/s
'HQ-PXY-Inet-1#Traffic-data#traffic_out' metric value is: 277.82 MB/s
'HQ-PXY-Inet-2#Traffic-data#traffic_in' metric value is: 906.22 KB/s
'HQ-PXY-Inet-2#Traffic-data#traffic_out' metric value is: 301.15 KB/s
```

