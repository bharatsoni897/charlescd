# Setting your metrics

## Register your metrics provider

### Istio Configuration

Metrics are related to circle requests, they are quantified and exposed by Istio, so it is necessary configure it to get information about each circle.

_As métricas relacionadas às requisições de cada círculo são quantificadas e expostas pelo Istio, por isso é necessário configurá-lo para que se tenha informações referentes à cada círculo._

{% hint style="info" %}
If you want to learn more about Istio's telemetry, check out their [**documentation**](https://istio.io/docs/tasks/observability/metrics/)**.**

  Se deseja entender um pouco mais sobre a telemetria do Istio, recomendamos que consulte a [doc oficial](https://istio.io/docs/tasks/observability/metrics/).
{% endhint %}

To configure your Istio, it is necessary to enable it, so it will be able to show metrics and then you have to configure to show Charles' metrics. 

If your Istio is not enabled to show metrics, follow the next steps:

 _Para configurar seu Istio é necessário habilitá-lo para expor métricas, e configurá-lo para expor as métricas do charles._

_Se seu Istio não está habilitado para expor métricas, siga os seguintes passos:_

{% hint style="warning" %}
The configuration below refers to Istio's 1.5 version.
{% endhint %}

Create a file named  **telemetry.yaml  with the following content:** 

```yaml
apiVersion: install.istio.io/v1alpha2
kind: IstioControlPlane
spec:
  values:
    prometheus:
      enabled: false
    telemetry:
      v1:
        enabled: true
      v2:
        enabled: false
```

Run the command below:

```bash
$ istioctl manifest apply -f telemetry.yaml
```

{% hint style="warning" %}
To run the command above, it is necessary to have configured the **istioctl**, if you haven't click [**here**](https://istio.io/docs/setup/getting-started/#download). 
{% endhint %}

To show the metrics related to Charles, you have to run the command: 

```bash
$ kubectl apply -f path/your-metrics-config.yaml
```

{% hint style="warning" %}
The file **your-metrics-config.yaml** that has been used must refer to the tool you use.
{% endhint %}

The files for configuration can be found below: 

{% tabs %}
{% tab title="Prometheus" %}
```yaml
# Configuration for request count metric instance
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata:
  name: charlesrequesttotal
  namespace: istio-system
spec:
  compiledTemplate: metric
  params:
    value: "1"
    dimensions:
      source: source.workload.name | "unknown"
      destination_pod: destination.workload.name | "unknown"
      destination_host: request.host | "unknown"
      destination_component: destination.labels["app"] | "unknown"
      circle_id: request.headers["x-circle-id"] | "unknown"
      circle_source: request.headers["x-circle-source"] | "unknown"
      response_status: response.code | 200
    monitoredResourceType: '"UNSPECIFIED"'
---
# Configuration for response duration metric instance
apiVersion: config.istio.io/v1alpha2
kind: instance
metadata: 
  name: charlesrequestduration
  namespace: istio-system
spec: 
  compiledTemplate: metric
  params: 
    value: response.duration | "0ms"
    dimensions:
      source: source.workload.name | "unknown"
      destination_pod: destination.workload.name | "unknown"
      destination_host: request.host | "unknown"
      destination_component: destination.labels["app"] | "unknown"
      circle_id: request.headers["x-circle-id"] | "unknown"
      circle_source: request.headers["x-circle-source"] | "unknown"
      response_status: response.code | 200
    monitoredResourceType: '"UNSPECIFIED"'
---     
# Configuration for a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: handler
metadata:
  name: charleshandler
  namespace: istio-system
spec:
  compiledAdapter: prometheus
  params:  
    metrics:
    - name: charles_request_total # Prometheus metric name
      instance_name: charlesrequesttotal.instance.istio-system
      kind: COUNTER
      label_names:
      - source
      - destination_pod
      - destination_host
      - destination_component
      - circle_id
      - circle_source
      - response_status
    - name: charles_request_duration_seconds # Prometheus metric name
      instance_name: charlesrequestduration.instance.istio-system
      kind: DISTRIBUTION
      label_names:
      - source
      - destination_pod
      - destination_host
      - destination_component
      - circle_id
      - circle_source
      - response_status
      buckets:
        explicit_buckets:
          bounds:
          - 0.01
          - 0.025
          - 0.05
          - 0.1
          - 0.25
          - 0.5
          - 0.75
          - 1
          - 2.5
          - 5
          - 10
---
# Rule to send metric instances to a Prometheus handler
apiVersion: config.istio.io/v1alpha2
kind: rule
metadata:
  name: charlesprom
  namespace: istio-system
spec:
  actions:
  - handler: charleshandler
    instances:
    - charlesrequesttotal
    - charlesrequestduration
```
{% endtab %}
{% endtabs %}

### Configuring your metrics tool

After you finish your Istio configuration it is necessary configure your metrics tool.

The first step is select the right tool, so Charles will be able to read. 

_Após feita a configuração do Istio é preciso configurar sua ferramenta para ser capaz de ler as métricas expostas._

_O primeiro passo é selecionar qual das ferramentas aceitas pelo Charles que você utiliza._

{% hint style="danger" %}
Charles until this moment only supports Prometheus as a metric tool, we are working on to bring others along the way.
{% endhint %}

{% hint style="info" %}
If the tool you use it isn't accepted yet, feel free to [**make a suggestion**](https://github.com/ZupIT/charlescd/issues) or make your own implementation and [**contribute with us**](https://github.com/ZupIT/charlescd/blob/master/CONTRIBUTING.md). Make our community grow more each day.

_Caso a ferramenta que você utilize não seja aceita ainda, fique à vontade de_ [_sugerir para nós_](https://github.com/ZupIT/charlescd/issues)_, ou faça sua implementação e_ [_contribua conosco_](https://github.com/ZupIT/charlescd/blob/master/CONTRIBUTING.md)_. Faça nossa comunidade crescer cada vez mais._ 
{% endhint %}

{% tabs %}
{% tab title="Prometheus" %}
Prometheus is an open-source systems monitoring and alerting toolkit. It is the main monitoring recommendation on [Cloud Native Computing Foundation](https://cncf.io/).

{% hint style="info" %}
If you want to know more about Prometheus, check it out [their documentation](https://prometheus.io/).
{% endhint %}

In order to Prometheus to be able to read and store metrics data, we must configure it. 

It is necessary to add the job below so it will read Istio's generated metrics. 

_Para o Prometheus conseguir ler e armazenar os dados das métricas que configuramos à pouco, é preciso configurá-lo._

_É preciso adicionar o job abaixo para que ele consiga ler as métricas geradas pelo Istio._

{% hint style="warning" %}
It is important to remember that all these configuration considers that your Prometheus is on the same Kubernete cluster as your Istio and the rest of your applications. 
{% endhint %}

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 1m
scrape_configs:
- job_name: istio-mesh
  scrape_interval: 15s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - istio-system #The namespace where your Istio is installed
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    separator: ;
    regex: istio-telemetry;prometheus
    replacement: $1
    action: keep
```

{% hint style="warning" %}
Stay tuned about the configuration **namespaces**, the configured value must be the same installed on your Istio.
{% endhint %}
{% endtab %}
{% endtabs %}

## 



