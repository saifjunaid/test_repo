![](.//media/rId20.png)


**Graylog Alerts from Logfiles**

Design Document

Revision History
----------------
| Version 	| Date          	| Author        	| Reviewer          	| Comments                                                          	|
|---------	|---------------	|---------------	|-------------------	|-------------------------------------------------------------------	|
| 0.1     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Initial Release                                                   	|
| 0.2     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Updates to input types and diagrams                               	|
| 0.3     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Added collector sidecar details and updated diagrams              	|
| 0.4     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Updated pipeline details and graylog local instance configuration 	|
| 0.5     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Removed duplicate sections                                        	|
| 0.6     	| December 2018 	| Anand S Katti 	| Prathap Thammanna 	| Updated correct message processing order for pipeline and streams 	|

> © 2018 by RadiSys Corporation. All rights reserved.
>
> Radisys, Network Service-Ready Platform, Quick!Start, TAPA, Trillium,
> Trillium+plus, Trillium Digital Systems, Trillium On Board, TAPA, and
> the Trillium logo are trademarks or registered trademarks of RadiSys
> Corporation. All other trademarks, registered trademarks, service
> marks, and trade names are the property of their respective owners.

REST

Contents
========

[Revision History 2](#revision-history)

[Chapter 1: Preface 5](#preface)

[1.1. Objective 5](#objective)

[1.2. Scope 5](#scope)

[Chapter 2: Overview 5](#overview)

[2.1. High level Log processing overview
5](#high-level-log-processing-overview)

[2.2. Detailed Logfiles processing 6](#detailed-logfiles-processing)

[2.2.1. Configuration and log processing in Graylog Local
7](#configuration-and-log-processing-in-graylog-local)

[2.2.1.1. INPUT Creation 7](#input-creation)

[2.2.1.2. OUTPUT Creation 9](#output-creation)

[2.2.1.3. Creation of Streams 9](#creation-of-streams)

[2.2.1.4. Pipeline processing of logfiles in stages -- Data Enriching
9](#pipeline-processing-of-logfiles-in-stages-data-enriching)

[2.2.1.5. Graylog Collector Sidecar Configuration
10](#graylog-collector-sidecar-configuration)

[2.2.2. Configuration of Graylog Central and Alerts Generation
11](#configuration-of-graylog-central-and-alerts-generation)

[2.2.3. Script for Configuring Graylog local and central instance
12](#script-for-configuring-graylog-local-and-central-instance)

[2.3. Container Networking for graylog local and central instance
13](#container-networking-for-graylog-local-and-central-instance)

[Appendix A: 14](#section)

[A.1. Sample graylog-rules.yml file 14](#sample-graylog-rules.yml-file)

[Appendix B: Graylog REST APIs 17](#graylog-rest-apis)

[B.1. REST APIs for getting Alert List
17](#rest-apis-for-getting-alert-list)

[B.2. Configuration of graylog sidecar in graylog local server
18](#configuration-of-graylog-sidecar-in-graylog-local-server)

[B.3. Installation and Configuration of collector sidecar in the targets
20](#installation-and-configuration-of-collector-sidecar-in-the-targets)

Figures
=======
[**Figure 1: High level Logfile Processing**
6](#Figure-1:-High-level-Logfile-Processing)

[**Figure 2: Detailed log flow in graylog**
7](#Xc69978e2e0be8b4143735e08adbbc66ce40214b)

[**Figure 3: Container Networking**
13](#X47ca25d5057fa79fadc09b7028e9790d068cf47)

Preface
=======

Objective
=========

The objective of this design document is to explain stage wise log files
processing inside the graylog. Includes details of enriching the content
of log entries, segregation of logs, generation of alerts from logfiles
including the configuring of graylog collector sidecar for log files
collection and configuration.

Graylog is a widely-used log management solution from open source
community. It runs as a microservice inside a container on POD MGMT
VMHOST in EMS setup.

Another instance of graylog node runs on EMS VMHOST as Graylog Central
Node.

Scope
=====

The scope of the design is limited to graylog configuration for
receiving the logfiles, processing, forwarding the filtered log messages
and raising alert notifications from the filtered log streams based on
the matching alert conditions.

Processing of graylog alert notifications if configured to send to
webhook is out of scope.

Overview
========

High level Log processing overview
==================================

There are two graylog nodes in the system, one running in the local pod
as "graylog local" and other running outside the pod as "graylog
central". The pod local graylog node receives the log messages at the
input and enriches the messages by modifying each message by adding new
details to the received messages.

Following diagram shows the flow of logfile entries across the graylog
nodes

![](.//media/rId29.png)


[]{#Xcbb8dd3dcb8b8f912b1b89de5551cca00583f49 .anchor}**Figure** **1:
High level Logfile Processing**

For ex: appending new keys (ex: pod\_name) to mark the messages for
further processing into a separate stream.

The enhanced messages are further segregated into different streams by
using the filter rules. These filtered messages are then forwarded to
the central graylog node outside the pod.

The central graylog node receives forwarded log entries at the input and
applies another level of filtering based on stream rules and makes the
filtered messages available in specific dedicated streams.

*Note: rules cannot be created on default stream "All messages". Rules
can only be created on any other newly created streams, whereas Alerts
can be created directly on stream "All messages"*

Several alert conditions can be created on these streams to raise
alerts, if those alert conditions are satisfied then alert notifications
are triggered to configured destinations (emails or webhook or etc)

Resolved and Unresolved Alerts can be seen in the Alerts tab of graylog
central node.

EMS can also read the resolved/un-resolved alerts from graylog central
node for further display or analysis.

Detailed Logfiles processing
============================

The logfile processing setup includes two graylog nodes along with an
EMS. As shown in **Figure 2: Detailed log flow in graylog** both the
graylog nodes are configured using **A.1** **Sample graylog-rules.yml
file**. This configuration file is split in two parts one for each
graylog nodes. The configuration parameters include important fields
that are needed to configure the graylog instances to filter messages
and generate alerts.

Following diagram shows the detailed design for graylog configuration
and logfiles processing

![](.//media/rId32.png)


[]{#Xc69978e2e0be8b4143735e08adbbc66ce40214b .anchor}**Figure** **2:
Detailed log flow in graylog**

Any logfiles from the targets can be transferred into the graylog local
instance by using filebeat packaged with graylog collector-sidecar
installer binary.

In Graylog version 2.4, a collector sidecar plugin is installed by
default. The functionality of this plugin is to monitor the change in
the filebeat configuration in the graylog and reconfigure filebeat
instances running on the target.

collector sidecar package needs to be installed on all the targets. It
interacts with the graylog local server over REST interface, to fetch
the Filebeat configuration file, whenever there is change in the
collector configuration on graylog local instance.

Refer section **Appendix B:Graylog REST APIs** for configuring collector
sidecar.

Every log entry in the logfiles **must** contain "**module\_name" key**
with **service name** as its **value.**

Configuration and log processing in Graylog Local
-------------------------------------------------

Configuration of graylog local instance includes configuration of
following graylog sub modules

### INPUT Creation

Graylog inputs are the endpoints created to receive data input into the
graylog system.

Different endpoints with different protocol are supported such as TCP,
GELF, syslog, beats, UDP etc.

In the EMS use case for metrics notification and log files collection,
two such inputs are created.

#### BEATS INPUT

A BEATS **INPUT** on **port** **5044** is created to receive log files
from filebeat instances running on target machines -- source of log
files. Configuration of filebeat is handled by the graylog collector
sidecar module which is installed as a separate package as explained in
section:B.1REST APIs for getting Alert List

#### GELF HTTP INPUT

A GELF HTTP is created for collecting the metric notifications from
prometheus alertmanager on **port 12201. **

### OUTPUT Creation

Graylog output contains IP Address and port details of the target
graylog node, where the filtered messages are forwarded. In the default
configuration, the filtered messages from AUTH stream are forwarded to
the destination host **graylog central node** on destination port
**12201. **

Accordingly, **GELF TCP INPUT** on graylog central node must be
available on **port 12201.**

### Creation of Streams

Streams are created to segregate incoming messages into dedicated
channels called as streams. Multiple streams can be created to support
multiple streams of filtered data.

In the default configuration on graylog local, an **AUTH** stream is
created which contains all the SSH Authentication Log messages.

One stream rule is created on **AUTH** stream, this rule ensures each
message **contains** value string "**AUTH"** in the field
**"module\_name"** in the **message** itself. If this rule matches the
message, then that message is moved into the **AUTH** stream. In the
default configuration, default "**Default index set" (graylog\_0)** is
used**,** hence no copy of the message is made when making it available
in the **"AUTH"** stream.

### Pipeline processing of logfiles in stages -- Data Enriching

In the default configuration, a single pipeline
"**LOG\_ENRICH\_PINELINE"** is created with a single stage 0, with one
rule in it. This pipeline is attached to "All messages" stream. Any
changes done in the pipelines are visible in all the streams including
"All messages".

The functionality of this pipeline is to add POD\_NAME filed (if
missing) to every message.

Above functionality is achieved by the following pipeline rules.

For ex: following pipeline rule adds the POD\_NAME field to every
message if that field is missing.

rule \"has pod\_name field\"

when

(has\_field(\"pod\_name\") \|\| has\_field(\"POD\_NAME\")) == false

then

set\_field(\"POD\_NAME\", \"POD-928\");

end

### Graylog Collector Sidecar Configuration

Graylog local instance collector configuration should be configured with
beats input/output configuration as described in previous sections.

It is assumed that graylog collector sidecar installer binary on the
target hosts is installed and configured ( ***see*** ***B.3***
**Installation and Configuration of collector sidecar in** the
targets**)** so that the **/var/log/auth.log** file from SERVERS (MGMT
VMHOST or CTRL VMHOST) are shipped into the graylog local instance.

Any changes in the collector configuration in the graylog local
instance, is periodically fetched by graylog collector sidecar running
on all the targets -- which restarts Filebeat with updated
configuration.

Additional sidecar configurations can be created with different tag and
logfile to export it to graylog local instance.

Configuration of Graylog Central and Alerts Generation
------------------------------------------------------

Graylog central instance receives messages from graylog local instance.
These filtered messages are segregated in to dedicated pre-configured
streams. Alerts and Notifications are further configured on these
streams.

In the default configuration for Graylog central instance, a **GELF TCP
INPUT** on **port 12201** is created to receive the forwarded filtered
log messages.

To further filter only the SSH failure messages from the received
messages, an **"AUTH"** stream is created, like the one created in
graylog local. This stream contains an additional rule to match every
message with string "authentication failure". This is useful for
creating alerts and notifications on these specific streams.

In the default configuration, an alert is raised, if **more than 1
"authentication failure"** log messages for any node/service is received
in the 1-minute time range. This alert of type **"Message Count Alert
Condition" on stream "AUTH".**

Graylog Central instance UI has "Alerts" tab which shows the resolved
and unresolved alerts received along with details analysis of alerts
duration, and events responsible for creation of the alert.

EMS or any entity can read these alerts from graylog central instance
with REST APIs.

If desired this alert can be forwarded as notification to the configured
HTTP Endpoint -- webhook.

Script for Configuring Graylog local and central instance
---------------------------------------------------------

To ease the configuration of graylog local and central instance, a
configuration file is made available in
**pod-config/graylog-config/graylog-rules.yml** directory on pod-config
GitLab project.

Configuration of both graylog nodes is automated with scripts using the
information from the graylog-rules.yml file.

A sample graylog configuration YAML file as available in **A.1Sample
graylog-rules.yml file**.

Graylog configuration script and configuration file is available in
project pod-config on gitlab

\$ git clone <https://gitlab.dol.telekom.de/Access40/pod-config>

\$ ls graylog-config/graylog-rules.yml

\$ ls graylog-init/

\$ ConfigureGraylog.py GraylogConstants.py GraylogUtils.py ParseYaml.py

**graylog-config/graylog-rules.yml** is updated with proper comments for
each of the parameters to make it easier for updating required fields.

If all the graylog instances are up and running, then run
ConfigureGraylog.py script to complete the configuration of the graylog
local and central instance.

\$ python3 ConfigureGraylog.py

Script will exit after **3 minutes** with error if graylog instances are
not up and running.

Container Networking for graylog local and central instance
===========================================================

All the containers are interconnected to each other and to the host via
Docker Bridge "pod-net". Pod-net bridge is associated with the subnet
172.20.20.0/24 and assigned an IP Address 172.20.20.1. All the
containers are given static IP addresses in the same subnet
172.20.20.0/24.

The following **Figure 3: Container Networking**, shows the containers
inter networking along with the IP Address of the containers.

![](.//media/rId46.png)


[]{#X47ca25d5057fa79fadc09b7028e9790d068cf47 .anchor}**Figure** **3:
Container Networking**

Sample graylog-rules.yml configuration file is used to configure graylog
nodes (local and central) with inputs, outputs, alerts, notifications
etc.

Sample graylog-rules.yml file 
==============================

\-\--

\#Local graylog configuration

graylog\_podlocal:

\#Host ip or DNS name

GRAYLOG\_HOSTNAME: \'172.27.170.203\'

\# pipeline configuration

pipeline:

\- name: \"LOG\_ENRICH\_PIPELINE\"

rules:

\- name: \"has auth content\"

source: \|

rule \"has auth content\"

when contains(to\_string(\$message.file), \"/var/log/auth.log\", true)

then set\_field(\"MODULE\_NAME\", \"AUTH\");

end

\- name: \"has pod\_name field\"

source: \|

rule \"has pod\_name field\"

when (has\_field(\"pod\_name\") \|\| has\_field(\"POD\_NAME\")) == false

then set\_field(\"POD\_NAME\", \"POD-928\");

end

\- name: \"has syslog content\"

source: \|

rule \"has syslog content\"

when contains(to\_string(\$message.file), \"/var/log/syslog\", true)

then set\_field(\"MODULE\_NAME\", \"SYSLOG\");

end

\#side car collector congfiguration

collector\_sidecar:

\- name: \'SSH\_AUTH\_LOG\'

tags:

\- \"SSH\_AUTH\_LOG\_TAG\"

\# out of the collector side car

output:

name: \"AUTH\_BEAT\_OUTPUT\"

backend: \"filebeat\"

type: \"logstash\"

properties:

hosts:

\- \"172.27.170.203:5044\"

\#input of the collector side car

input:

name: \"AUTH\_BEAT\_INPUT\"

backend: \"filebeat\"

type: \"file\"

properties:

paths:

\- \'/var/log/auth.log\'

\# input configuration for graylog

graylog\_input:

\- title: \'BEATS INPUT\'

type: \'org.graylog.plugins.beats.BeatsInput\'

port: \'5044\'

enable\_cors: \'true\'

\- title: \'GELF HTTP\'

type: \'org.graylog2.inputs.gelf.http.GELFHttpInput\'

port: \'12201\'

enable\_cors: \'true\'

graylog\_output:

\- title: \'GELF\_TCP\'

type: \'org.graylog2.outputs.GelfOutput\'

destination\_port: \'12201\'

destination\_host: \'172.27.170.141\'

protocol: \'TCP\'

\# streams configuration for graylog

streams:

\- title: \'AUTH\' \# AUTH,SYSLOG,LCM,VOLTHA,PPA,PODGW etc..

description: \'SSH Authentication LOGS\'

output:

name: \"GELF\_TCP\"

stream\_rule:

\# \"module\_name\" can be
message,alarm\_name,facility,full\_message,level,

\# element\_type,element\_name etc..

\- field: \'module\_name\'

\# \'contain\',\'match exactly\',\'match regular expression\',\'greater
than\',

\#\'smaller than\',\'field presence\',\'always match\'

type: \'always match\'

value: \'AUTH\'

\- field: \'message\'

type: \'contain\'

value: \'authentication failure\'

graylog\_central:

GRAYLOG\_HOSTNAME: \'172.27.170.141\'

graylog\_input:

\- title: \'GELF TCP\'

type: \'org.graylog2.inputs.gelf.tcp.GELFTCPInput\'

port: \'12201\'

enable\_cors: \'true\'

streams:

\- title: \'AUTH\'

description: \'SSH Authentication LOGS\'

stream\_rule:

\- field: \'module\_name\'

type: \'contain\'

value: \'AUTH\'

\- field: \'message\'

type: \'contain\'

value: \'authentication failure\'

alerts:

alert\_condition:

\- title: \'SSH\_AUTHENTICATION\_FAILURE\_ALERT\'

type: \'Field Content Alert Condition\'

parameters:

time\_range: 2 \# time in minutes

threshold\_type: \'more than\' \# \'less than\'

threshold: 1 \# value which triggers an alert if crossed

grace\_period: 2 \# no. of minutes after which an alert condition is
resolved

message\_backlog: 1 \# no. of messages to be included in alert
notifications

field: \'element-type\'

value: \'OLT\'

\- title: \'SSH\_AUTHENTICATION\_FAILURE ALERT\'

type: \'Message Count Alert Condition\'

parameters:

time\_range: 2 \# time in minutes

threshold\_type: \'more than\' \# \'less than\'

threshold: 1 \# value which triggers an alert if crossed

grace\_period: 2 \# no. of minutes after which an alert condition is
resolved

message\_backlog: 1 \# no. of messages to be included in alert
notifications

alert\_notification:

\- type: \'org.graylog2.alarmcallbacks.HTTPAlarmCallback\'

title: \'SSH Failure Alert Notification\'

configuration:

url: \'http://172.27.170.167:9000/alerts/\'

Graylog REST APIs 
==================

REST APIs for getting Alert List
================================

Following are some of the REST APIs that can be used to fetch the list
of resolved alerts from central graylog node.

**Get all the enabled streams:**

API: GET /streams/enabled

In the output look for **value:"AUTH"** streams**,** and corresponding
**"streamd\_id"** field contains the stream id.

For ex: \"stream\_id\": \"5c1a0e85f77f5f0001ff1434\"

**Get all the alerts (Resolved and Un Resolved) on this stream:**

GET /streams/{streamId}/alerts

Gives the list of all alerts. Those alerts which have empty
**"resolved\_at"** filed are un-resolved alerts.

For ex: Following snippet shows the REST API response for GET streams
alerts

{

\"total\": 21,

\"alerts\": \[

{

\"id\": \"5c1a6314f77f5f000132b5e0\",

\"description\": \"Stream had 2 messages in the last 1 minutes with
trigger condition more than 1 messages. (Current grace time: 1
minutes)\",

\"condition\_id\": \"2a79e11f-c57a-4f17-aae9-cdbaa33facd4\",

\"stream\_id\": \"5c1a0e85f77f5f0001ff1434\",

\"condition\_parameters\": {

\"backlog\": 1,

\"repeat\_notifications\": false,

\"grace\": 1,

\"threshold\_type\": \"MORE\",

\"threshold\": 1,

\"time\": 1

},

\"triggered\_at\": \"2018-12-19T15:26:12.689Z\",

\"resolved\_at\": \"2018-12-19T15:27:12.682Z\",

\"is\_interval\": true

},

For Testing or validation, one can use the REST API BROWSER on graylog
central node itself.

For ex: <http://GRAYLOG-CENTRAL:9080/api/api-browser> provides an
extensive list of API to run on the live graylog instance

Configuration of graylog sidecar in graylog local server
========================================================

Graylog collector plugin on the graylog local should be configured in
the following order.

1)  Create new collector configuration with configuration name
    ="SSH\_AUTH\_LOG" and tag="SSH\_AUTH\_LOG\_TAG"

-   **REST API:** POST
    /api/plugins/org.graylog.plugins.collector/configurations

    **Request Body:**

    {

    \"tags\":\[

    \"SSH\_AUTH\_LOG\_TAG\"

    \],

    \"inputs\":\[

    \],

    \"outputs\":\[

    \],

    \"snippets\":\[

    \],

    \"name\":\"SSH\_AUTH\_LOG\"

    }

    **Expected Response Body:**

    {

    \"tags\": \[

    \"SSH\_AUTH\_LOG\_TAG\"

    \],

    \"inputs\": \[\],

    \"outputs\": \[\],

    \"snippets\": \[\],

    \"id\": \"5c1cbfb1f77f5f00015a9a5d\",

    \"name\": \"SSH\_AUTH\_LOG\"

    }

2)  Create OUTPUT with OUTPUT name="AUTH\_BEAT\_OUTPUT" and type of
    backend="filebeat" and type="logstash" on
    host=GRAYLOG\_LOCAL\_SERVER \_IP:5044

-   **REST API:**
    /api/plugins/org.graylog.plugins.collector/configurations/{id}/outputs

    For {Id} field, use the value of id from the previous command
    Response Body, as shown highlighted.

    **Request Json Body:**

    {

    \"name\": \"AUTH\_BEAT\_OUTPUT\",

    \"backend\": \"filebeat\",

    \"type\": \"logstash\",

    \"properties\": {

    \"hosts\": \"\[\'172.27.170.203:5044\'\]\"

    }

    }

    **Expected Response**

    No output:

    **Expected Response code:** 202

3)  Make GET call to show collector configuration details, form the
    output, note down the output id, which will be used in the
    subsequent calls to create collector input.

-   **REST API:** GET
    /plugins/org.graylog.plugins.collector/configurations/{id}

    For {Id} field, use the value of id from the 1^st^ command Response
    Body, as shown highlighted.

    **Request Json Body:**

    Id: 5c1cbfb1f77f5f00015a9a5d

    **Response Body:**

    {

    \"tags\": \[

    \"SSH\_AUTH\_LOG\_TAG\"

    \],

    \"inputs\": \[\],

    \"outputs\": \[

    {

    \"backend\": \"filebeat\",

    \"type\": \"logstash\",

    \"name\": \"AUTH\_BEAT\_OUTPUT\",

    \"properties\": {

    \"hosts\": \"\[\'172.27.170.203:5044\'\]\"

    },

    \"output\_id\": \"5c1cc3a2f77f5f00015a9eae\"

    }

    \],

    \"snippets\": \[\],

    \"id\": \"5c1cbfb1f77f5f00015a9a5d\",

    \"name\": \"SSH\_AUTH\_LOG\"

    }

    Note the highlighted "output\_id"

4)  Create a configuration INPUT, forward to parameter is the value of
    the output\_id field.

> **REST API:** POST
> /plugins/org.graylog.plugins.collector/configurations/{id}/inputs
>
> **Parameters:**
>
> **ID:** 5c1cbfb1f77f5f00015a9a5d( Configuration ID)
>
> **Request Json Body:**

{

\"name\": \"AUTH\_BEAT\_INPUT\",

\"backend\": \"filebeat\",

\"type\": \"file\",

\"properties\": {

\"paths\": \"\[\'/var/log/auth.log\'\]\",

},

\"forward\_to\": \"5c1cc3a2f77f5f00015a9eae\"

\#( This is the output\_id received in the previous command)

}

Installation and Configuration of collector sidecar in the targets
==================================================================

On all the targets from where any logfiles need to be sent to the
graylog server, this package needs to be installed.

1)  **Graylog collector sidecar** binary package for Debian can be
    downloaded from
    <https://github.com/Graylog2/collector-sidecar/releases/download/0.1.7/collector-sidecar_0.1.7-1_amd64.deb>

2)  Install the package

-   It installs package filebeat also along with a set of configuration
    file include
    **"/etc/graylog/collector-sidecar/collector\_sidecar.yml"**

> sudo dpkg -i collector-sidecar\_0.1.7-1\_amd64.deb
>
> sudo graylog-sidecar -service install

3)  Edit /etc/graylog/collector-sidecar/collector\_sidecar.yml and
    following two fields. The tag field is used by collector sidecar to
    fetch the beats configuration from the graylog-server from the match
    tag.

> server\_url: <http://GRAYLOG_LOCAL_IP:9080/api/>
>
> tags:

-   SSH\_AUTH\_LOG\_TAG

4)  start the collector sidecar

> sudo systemctl start collector-sidecar

5)  verify if its running

> sudo systemctl status collector-sidecar

6)  If its successfully running, then this node/target should be visible
    in the graylog local instance

-   Under the URI: <http://GRAYLOG_LOCAL_IP:9080/system/collectors>

