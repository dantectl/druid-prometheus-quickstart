# druid-prometheus-quickstart

Instructions on how to monitor Apache Druid using Prometheus and New Relic

*This guide is written for a local installation of Apache Druid for demonstration purposes. For production deployments, the overall process should be similar, however you may notice some variation in the directory structure etc.* 

## Requirements
* [New Relic Account](https://newrelic.com/signup/) - *free to sign up*
* [Prometheus Server](https://prometheus.io/docs/prometheus/latest/getting_started/)
* [Apache Druid Server](https://druid.apache.org/docs/latest/tutorials/index.html)
* [Prometheus Extension for Druid](https://druid.apache.org/docs/latest/development/extensions.html#loading-extensions)


## High Level Process
* Update runtime.properties files in Druid runtimes
* Configure Prometheus server to point to Druid endpoints
* Configure Prometheus to point to New Relic Remote Write Endpoint
* Deploy Dashboard in New Relic and configure alerts


## Getting Started
In your Druid deployment, there are likely six different processes that you’ll want to monitor. To view the metrics for these processes, you’ll need to configure a druid.emitter for each. This will expose the metrics and allow them to be scraped and forwarded by Prometheus. These are the six processes:

* Common
* Broker
* Coordinator-Overlord
* Historical
* MiddleManager
* Router


## Common Runtime
For starters, in order to configure monitoring for Druid, you’ll need to expose the metrics using the `prometheus.emitter`. We’ll get this installed later in the doc, but for now, let’s add the necessary lines in our config files. 

In the `_common` directory, open the `common.runtime.properties` file and find the extensions list. Then, add `prometheus.emitter` at the end, see below for reference:
```
cd ~/apache-druid-24.0.0/conf/druid/single-server/micro-quickstart
```
```
druid.extensions.loadList=["druid-hdfs-storage", "druid-kafka-indexing-service", "druid-datasketches", "druid-multi-stage-query", "prometheus-emitter"]
```


## All Other Runtimes
In the **Broker, Coordinator-Overlord, Historical, MiddleManager, and Router** folders, open the `runtime.properties` file and paste the following snippet at the end of each:
```
# Monitoring
druid.monitoring.monitors=["org.apache.druid.java.util.metrics.JvmMonitor"]
druid.emitter=prometheus
druid.emitter.logging.logLevel=info
druid.emitter.prometheus.strategy=exporter
druid.emitter.prometheus.port=19092
```


## Installing the Prometheus Emitter Extension
Now that you’ve configured the runtime properties with the necessary monitoring settings, you’ll want to install the prometheus extension. This includes a number of jar files that will be called when the server starts up. Run the following command from the install path where you have Apache Druid set up (typically `~/apache-druid-24.0.0/`):
```
cd ~/apache-druid-24.0.0/
```
```
java \
  -cp "lib/*" \
  -Ddruid.extensions.directory="extensions" \
  -Ddruid.extensions.hadoopDependenciesDir="hadoop-dependencies" \
  org.apache.druid.cli.Main tools pull-deps \
  --no-default-hadoop \
  -c "org.apache.druid.extensions.contrib:prometheus-emitter:24.0.0"
```


## Configure Prometheus Scraping
Install and set up Prometheus following [their guide](https://prometheus.io/docs/prometheus/latest/getting_started/).  Next, you can follow [this guide](https://docs.newrelic.com/docs/infrastructure/prometheus-integrations/install-configure-remote-write/set-your-prometheus-remote-write-integration/) to get the remote_write URL to send metrics to New Relic - or you can use this link to leverage our guided installation. Here’s an example of what the prometheus.yml file could look like. 
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
remote_write:
- url: https://metric-api.newrelic.com/prometheus/v1/write?prometheus_server=prometheus-druid
  bearer_token: <YOUR-NR-LICENSE-KEY>

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus-druid"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:19090"]
      - targets: ["localhost:19091"]
      - targets: ["localhost:19092"]
      - targets: ["localhost:19093"]
      - targets: ["localhost:19094"]
      - targets: ["localhost:19095"]
```

## Configure Infrastructure and Log Monitoring

### Infrastructure Monitoring
New Relic has a [guided installation](https://one.newrelic.com/launcher/nr1-core.explorer?pane=eyJuZXJkbGV0SWQiOiJucjEtY29yZS5saXN0aW5nIn0=&cards[0]=eyJuZXJkbGV0SWQiOiJucjEtaW5zdGFsbC1uZXdyZWxpYy5ucjEtaW5zdGFsbC1uZXdyZWxpYyIsImFjdGl2ZUNvbXBvbmVudCI6IlZUU09FbnZpcm9ubWVudCIsInBhdGgiOiJndWlkZWQifQ==) to automatically install and configure New Relic’s Infrastructure Agent. This will allow you to see host level metrics for CPU, Memory, Storage etc. These will be vital for understanding the overall health of your Apache Druid server. Alternative to the guided install, you can instead [install and configure the agent manually](https://docs.newrelic.com/docs/infrastructure/install-infrastructure-agent/linux-installation/install-infrastructure-monitoring-agent-linux/). 

### Log Monitoring
New Relic’s Infrastructure automatically comes with a Log Forward installed. Open the log configuration file:
```
nano /etc/newrelic-infra/logging.d/logging.yml
```

Then, add the following line to the bottom of the file. Be sure the formatting lines up with the [example from the documentation](https://docs.newrelic.com/docs/logs/forward-logs/forward-your-logs-using-infrastructure-agent/#file):
```
  - name: druid-logs
    file: /home/<yourhomedir>/apache-druid-24.0.0/log/*.log
    attributes:
      logtype: apache-druid
```


You should now see logs for Apache Druid in the NR UI:
```
FROM Log SELECT * where logtype = 'apache-druid'
```

