# Prometheus Input Plugin

The prometheus input plugin gathers metrics from HTTP servers exposing metrics
in Prometheus format.

## Configuration

```toml
# Read metrics from one or many prometheus clients
[[inputs.prometheus]]
  ## An array of urls to scrape metrics from.
  urls = ["http://localhost:9100/metrics"]
  
  ## Metric version controls the mapping from Prometheus metrics into
  ## Telegraf metrics.  When using the prometheus_client output, use the same
  ## value in both plugins to ensure metrics are round-tripped without
  ## modification.
  ##
  ##   example: metric_version = 1; 
  ##            metric_version = 2; recommended version
  # metric_version = 1
  
  ## Url tag name (tag containing scrapped url. optional, default is "url")
  # url_tag = "url"
  
  ## Whether the timestamp of the scraped metrics will be ignored.
  ## If set to true, the gather time will be used.
  # ignore_timestamp = false
  
  ## An array of Kubernetes services to scrape metrics from.
  # kubernetes_services = ["http://my-service-dns.my-namespace:9100/metrics"]
  
  ## Kubernetes config file to create client from.
  # kube_config = "/path/to/kubernetes.config"
  
  ## Scrape Kubernetes pods for the following prometheus annotations:
  ## - prometheus.io/scrape: Enable scraping for this pod
  ## - prometheus.io/scheme: If the metrics endpoint is secured then you will need to
  ##     set this to 'https' & most likely set the tls config.
  ## - prometheus.io/path: If the metrics path is not /metrics, define it with this annotation.
  ## - prometheus.io/port: If port is not 9102 use this annotation
  # monitor_kubernetes_pods = true
  
  ## Get the list of pods to scrape with either the scope of
  ## - cluster: the kubernetes watch api (default, no need to specify)
  ## - node: the local cadvisor api; for scalability. Note that the config node_ip or the environment variable NODE_IP must be set to the host IP.
  # pod_scrape_scope = "cluster"
  
  ## Only for node scrape scope: node IP of the node that telegraf is running on.
  ## Either this config or the environment variable NODE_IP must be set.
  # node_ip = "10.180.1.1"
 
  ## Only for node scrape scope: interval in seconds for how often to get updated pod list for scraping.
  ## Default is 60 seconds.
  # pod_scrape_interval = 60
  
  ## Restricts Kubernetes monitoring to a single namespace
  ##   ex: monitor_kubernetes_pods_namespace = "default"
  # monitor_kubernetes_pods_namespace = ""
  # label selector to target pods which have the label
  # kubernetes_label_selector = "env=dev,app=nginx"
  # field selector to target pods
  # eg. To scrape pods on a specific node
  # kubernetes_field_selector = "spec.nodeName=$HOSTNAME"

  ## Scrape Services available in Consul Catalog
  # [inputs.prometheus.consul]
  #   enabled = true
  #   agent = "http://localhost:8500"
  #   query_interval = "5m"

  #   [[inputs.prometheus.consul.query]]
  #     name = "a service name"
  #     tag = "a service tag"
  #     url = 'http://{{if ne .ServiceAddress ""}}{{.ServiceAddress}}{{else}}{{.Address}}{{end}}:{{.ServicePort}}/{{with .ServiceMeta.metrics_path}}{{.}}{{else}}metrics{{end}}'
  #     [inputs.prometheus.consul.query.tags]
  #       host = "{{.Node}}"
  
  ## Use bearer token for authorization. ('bearer_token' takes priority)
  # bearer_token = "/path/to/bearer/token"
  ## OR
  # bearer_token_string = "abc_123"
  
  ## HTTP Basic Authentication username and password. ('bearer_token' and
  ## 'bearer_token_string' take priority)
  # username = ""
  # password = ""
  
  ## Specify timeout duration for slower prometheus clients (default is 3s)
  # response_timeout = "3s"
  
  ## Optional TLS Config
  # tls_ca = /path/to/cafile
  # tls_cert = /path/to/certfile
  # tls_key = /path/to/keyfile
  
  ## Use TLS but skip chain & host verification
  # insecure_skip_verify = false
```

`urls` can contain a unix socket as well. If a different path is required (default is `/metrics` for both http[s] and unix) for a unix socket, add `path` as a query parameter as follows: `unix:///var/run/prometheus.sock?path=/custom/metrics`

### Kubernetes Service Discovery

URLs listed in the `kubernetes_services` parameter will be expanded
by looking up all A records assigned to the hostname as described in
[Kubernetes DNS service discovery](https://kubernetes.io/docs/concepts/services-networking/service/#dns).

This method can be used to locate all
[Kubernetes headless services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).

### Kubernetes scraping

Enabling this option will allow the plugin to scrape for prometheus annotation on Kubernetes
pods. Currently, you can run this plugin in your kubernetes cluster, or we use the kubeconfig
file to determine where to monitor.
Currently the following annotation are supported:

* `prometheus.io/scrape` Enable scraping for this pod.
* `prometheus.io/scheme` If the metrics endpoint is secured then you will need to set this to `https` & most likely set the tls config. (default 'http')
* `prometheus.io/path` Override the path for the metrics endpoint on the service. (default '/metrics')
* `prometheus.io/port` Used to override the port. (default 9102)

Using the `monitor_kubernetes_pods_namespace` option allows you to limit which pods you are scraping.

Using `pod_scrape_scope = "node"` allows more scalable scraping for pods which will scrape pods only in the node that telegraf is running. It will fetch the pod list locally from the node's kubelet. This will require running Telegraf in every node of the cluster. Note that either `node_ip` must be specified in the config or the environment variable `NODE_IP` must be set to the host IP. ThisThe latter can be done in the yaml of the pod running telegraf:

```sh
env:
  - name: NODE_IP
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
 ```

If using node level scrape scope, `pod_scrape_interval` specifies how often (in seconds) the pod list for scraping should updated. If not specified, the default is 60 seconds.

### Consul Service Discovery

Enabling this option and configuring consul `agent` url will allow the plugin to query
consul catalog for available services. Using `query_interval` the plugin will periodically
query the consul catalog for services with `name` and `tag` and refresh the list of scraped urls.
It can use the information from the catalog to build the scraped url and additional tags from a template.

Multiple consul queries can be configured, each for different service.
The following example fields can be used in url or tag templates:

* Node
* Address
* NodeMeta
* ServicePort
* ServiceAddress
* ServiceTags
* ServiceMeta

For full list of available fields and their type see struct CatalogService in
<https://github.com/hashicorp/consul/blob/master/api/catalog.go>

### Bearer Token

If set, the file specified by the `bearer_token` parameter will be read on
each interval and its contents will be appended to the Bearer string in the
Authorization header.

## Usage for Caddy HTTP server

Steps to monitor Caddy with Telegraf's Prometheus input plugin:

* Download [Caddy](https://caddyserver.com/download)
* Download Prometheus and set up [monitoring Caddy with Prometheus metrics](https://caddyserver.com/docs/metrics#monitoring-caddy-with-prometheus-metrics)
* Restart Caddy
* Configure Telegraf to fetch metrics on it:

```toml
[[inputs.prometheus]]
#   ## An array of urls to scrape metrics from.
  urls = ["http://localhost:2019/metrics"]
```

> This is the default URL where Caddy will send data.
> For more details, please read the [Caddy Prometheus documentation](https://github.com/miekg/caddy-prometheus/blob/master/README.md).

## Metrics

Measurement names are based on the Metric Family and tags are created for each
label.  The value is added to a field named based on the metric type.

All metrics receive the `url` tag indicating the related URL specified in the
Telegraf configuration. If using Kubernetes service discovery the `address`
tag is also added indicating the discovered ip address.

## Example Output

### Source

```shell
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 7.4545e-05
go_gc_duration_seconds{quantile="0.25"} 7.6999e-05
go_gc_duration_seconds{quantile="0.5"} 0.000277935
go_gc_duration_seconds{quantile="0.75"} 0.000706591
go_gc_duration_seconds{quantile="1"} 0.000706591
go_gc_duration_seconds_sum 0.00113607
go_gc_duration_seconds_count 4
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 15
# HELP cpu_usage_user Telegraf collected metric
# TYPE cpu_usage_user gauge
cpu_usage_user{cpu="cpu0"} 1.4112903225816156
cpu_usage_user{cpu="cpu1"} 0.702106318955865
cpu_usage_user{cpu="cpu2"} 2.0161290322588776
cpu_usage_user{cpu="cpu3"} 1.5045135406226022
```

### Output

```shell
go_gc_duration_seconds,url=http://example.org:9273/metrics 1=0.001336611,count=14,sum=0.004527551,0=0.000057965,0.25=0.000083812,0.5=0.000286537,0.75=0.000365303 1505776733000000000
go_goroutines,url=http://example.org:9273/metrics gauge=21 1505776695000000000
cpu_usage_user,cpu=cpu0,url=http://example.org:9273/metrics gauge=1.513622603430151 1505776751000000000
cpu_usage_user,cpu=cpu1,url=http://example.org:9273/metrics gauge=5.829145728641773 1505776751000000000
cpu_usage_user,cpu=cpu2,url=http://example.org:9273/metrics gauge=2.119071644805144 1505776751000000000
cpu_usage_user,cpu=cpu3,url=http://example.org:9273/metrics gauge=1.5228426395944945 1505776751000000000
```

### Output (when metric_version = 2)

```shell
prometheus,quantile=1,url=http://example.org:9273/metrics go_gc_duration_seconds=0.005574303 1556075100000000000
prometheus,quantile=0.75,url=http://example.org:9273/metrics go_gc_duration_seconds=0.0001046 1556075100000000000
prometheus,quantile=0.5,url=http://example.org:9273/metrics go_gc_duration_seconds=0.0000719 1556075100000000000
prometheus,quantile=0.25,url=http://example.org:9273/metrics go_gc_duration_seconds=0.0000579 1556075100000000000
prometheus,quantile=0,url=http://example.org:9273/metrics go_gc_duration_seconds=0.0000349 1556075100000000000
prometheus,url=http://example.org:9273/metrics go_gc_duration_seconds_count=324,go_gc_duration_seconds_sum=0.091340353 1556075100000000000
prometheus,url=http://example.org:9273/metrics go_goroutines=15 1556075100000000000
prometheus,cpu=cpu0,url=http://example.org:9273/metrics cpu_usage_user=1.513622603430151 1505776751000000000
prometheus,cpu=cpu1,url=http://example.org:9273/metrics cpu_usage_user=5.829145728641773 1505776751000000000
prometheus,cpu=cpu2,url=http://example.org:9273/metrics cpu_usage_user=2.119071644805144 1505776751000000000
prometheus,cpu=cpu3,url=http://example.org:9273/metrics cpu_usage_user=1.5228426395944945 1505776751000000000
```
