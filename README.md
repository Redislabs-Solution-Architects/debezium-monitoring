# JMX Monitoring for Debezium Server
The connectors in Debezium Server provide JMX metrics that can be used for monitoring with a suitable tool, for example [JConsole](https://docs.oracle.com/javase/6/docs/technotes/guides/management/jconsole.html) or [Grafana](https://grafana.com/). See section Monitoring in the Debezium documentation for the various connectors, for example [here](https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-monitoring) for MySQL. This guide describes the steps required for setting up monitoring for Debezium Server.

## Java Options

JMX monitoring is enabled by passing some options to the Java Virtual Machine (JVM) . Specifically, these are:
```
-Dcom.sun.management.jmxremote.ssl=false 
-Dcom.sun.management.jmxremote.authenticate=false 
-Dcom.sun.management.jmxremote.port=<PORT_NUMBER> 
-Dcom.sun.management.jmxremote.rmi.port=<PORT_NUMBER>
-Dcom.sun.management.jmxremote.local.only=false 
-Djava.rmi.server.hostname=<HOSTNAME>
```

`<PORT_NUMBER>` is an available port on the machine running Debezium Server. This can be the same for `jmxremote.port` and `jmxremote.rmi.port`, but note that the JMX client connects through jmxremote.port.

`<HOSTNAME>` is the name of the Debezium Server machine, but for convenience it can be set to `0.0.0.0`.

The options are passed to the JVM through environment variable __*JAVA_OPTS*__. If you are running Debezium Server as a container, you also need to open the port(s) specified in the options. The Docker command for running Debezium Server looks like this:
```
docker run -d --name debezium -v $PWD/conf:/debezium/conf \
  -e JAVA_OPTS="-Dcom.sun.management.jmxremote.ssl=false \
                -Dcom.sun.management.jmxremote.authenticate=false \
                -Dcom.sun.management.jmxremote.port=1099 \
                -Dcom.sun.management.jmxremote.rmi.port=1099 \
                -Dcom.sun.management.jmxremote.local.only=false \
                -Djava.rmi.server.hostname=0.0.0.0" \
  -p 1099:1099 debezium/server:2.1.2.Final
```

## Monitoring through JConsole

Once Debezium Server is running, connect to the JVM through JConsole. In this example, JConsole and Docker are running on the same machine (*localhost*):

<img src="https://user-images.githubusercontent.com/116373419/231172750-051915a2-28ec-41f5-b75e-3c315449800d.png" height="400">

→ Connect

<img src="https://user-images.githubusercontent.com/116373419/231173036-c00e71e1-af93-45d4-ac35-dc906482be0c.png" height="200">

→ Insecure connection

<img src="https://user-images.githubusercontent.com/116373419/231173137-1db59f72-bfa2-42f2-93bc-1f450dfa91e9.png" height="800">

→ MBeans

<img src="https://user-images.githubusercontent.com/116373419/231173198-c6308acd-74ef-49a4-a730-913d8f0d1909.png" height="800">

The available MBeans will depend on the connector you are using with Debezium Server. The screenshot above shows the three MBeans for MySQL, but some connectors (for example PostgreSQL) only have two MBeans for snapshot and streaming.

## Monitoring through Grafana

If you want to monitor through Grafana, three additional components are needed:

* [JMX Exporter for Prometheus](https://github.com/prometheus/jmx_exporter)
* [Prometheus](https://prometheus.io/download/)
* [Grafana](https://grafana.com/grafana/download)

This guide assumes that all components are running on the local workstation.

### JMX Exporter for Prometheus

The JMX Exporter can be run as a standalone HTTP server, but the recommended method is to use it as a Java agent. If you are running Debezium Server as a container, follow these steps:

1. Copy the java agent jar file to the conf directory that contains the application.properties file for Debezium

   ```
   cp jmx_prometheus_javaagent-0.18.0.jar conf
   ```

2. In the conf directory create file config.yaml:
   ```
   startDelaySeconds: 0
   ssl: false
   lowercaseOutputName: false
   lowercaseOutputLabelNames: false
   rules:
   - pattern: "debezium.([^:]+)<type=connector-metrics, context=([^,]+), server=([^,]+), key=([^>]+)><>RowsScanned"
     name: "debezium_metrics_RowsScanned"
     labels:
       plugin: "$1"
       name: "$3"
       context: "$2"
       table: "$4"
   - pattern: "debezium.([^:]+)<type=connector-metrics, context=([^,]+), server=([^>]+)>([^:]+)"
     name: "debezium_metrics_$4"
     labels:
       plugin: "$1"
       name: "$3"
       context: "$2"
   ```
   
3. Add the java agent to the Docker run command:
   ```
   docker run -d --name debezium -v $PWD/conf:/debezium/conf \
     -e JAVA_OPTS="-javaagent:/debezium/conf/jmx_prometheus_javaagent-0.18.0.jar=12345:/debezium/conf/config.yaml" \
     -p 12345:12345 debezium/server:2.1.2.Final
   ```

Note the parameter for mapping port 12345. This can be any free port on your machine.

You can now view the Debezium metrics in your browser at address *http://localhost:12345*:

![image](https://user-images.githubusercontent.com/116373419/231174673-92b816fa-de5c-4024-8a7e-1251bcbe2c49.png)

### Prometheus

Add the JMX Exporter to file *prometheus.yml*:
```
global:
  scrape_interval:     15s 
  evaluation_interval: 15s 

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  # scrape JMX Metrics Exporter
  - job_name: debezium
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:12345"]
```

> The job name must be debezium, as this is hard-wired in the Grafana dashboard.

Restart Prometheus and verify the target for Debezium is available (*http://localhost:9090*, menu item *Status > Targets*):

![image](https://user-images.githubusercontent.com/116373419/231175002-0ca41a9c-5ea3-4ee9-afda-bdca464a8c54.png)

> Prometheus changes *localhost* or *127.0.0.1* in the endpoint URL to the machine’s hostname. If you want to access these URLs, simply add the hostname of the machine as an alias for localhost in */etc/hosts*.

### Grafana

Currrently, there are two sample Grafana dashboards available, one for MySQL and the other for Oracle. These are based on the dashboard for MySQL available in the [Debezium Examples](https://github.com/debezium/debezium-examples/tree/main/monitoring/debezium-grafana) repository. Note that this is a bit outdated and will not work well with the current version of Grafana (9.3.6 as of April 2023).

Download the monitoring dashboard for the Debezium MySQL or Oracle connector from this repository, import the JSON file into Grafana and connect the dashboard to your Prometheus data source.

Each dashboard has three sections:

<ins>Streaming</ins>

<img width="1508" alt="Grafana dashboard streaming section" src="https://user-images.githubusercontent.com/116373419/231205296-fc6d824e-3d22-4dff-bf0a-cc1c446a881a.png">

<ins>Snapshot</ins>

<img width="1506" alt="Grafana dashboard snapshot section" src="https://user-images.githubusercontent.com/116373419/231205567-5ca5fd51-4286-4cc4-ba42-8223d8296ccd.png">

<ins>Schema History</ins>

<img width="1511" alt="Grafana dashboard schema history section" src="https://user-images.githubusercontent.com/116373419/231204934-e1d8e6d1-32bb-49c6-b1ee-2e7a63c39cc9.png">

