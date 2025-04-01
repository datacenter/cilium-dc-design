---
title: Talos How To
layout: default
parent: Tetragon
nav_order: 2
---

{: .warning }
You will most likely have to customize the YAML Examples or CLI/Versions in this section to suit your needs

# Pre Requisite

It is expected that you have installed the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack). If you are installing in a **lab** and you want to quick config example that provide the stack running on Emphemeral Storage you can use these values:

{: .warning }
**Do not use for production**: This deploys an instance without Authentication that will accept ScrapingConfigs from any namespaces. How to deploy a production grade [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) is outside the scope of this design guide.


```yaml
defaultRules:
  rules:
    kubeProxy: false
    windows: false
grafana:
  persistence:
    enabled: false
  service:
    service:
    enabled: true
    type: LoadBalancer  # You might need to change this to use an ingress controller instead
kubeProxy:
  enabled: false
prometheus:
  prometheusSpec:
    ### find any pod, scrapeConfig or servicemonitor in the cluster and don't worry about how they are labeled so everything should be scraped both cluster metrics as well as tetragon.
    scrapeConfigSelectorNilUsesHelmValues: false
    scrapeConfigSelector: {}
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    podMonitorSelector: {}
    podMonitorSelectorNilUsesHelmValues: false
    logLevel: info
kubeApiServer:
  tlsConfig:
    insecureSkipVerify: true
```

# Installing Tetragon

Tetragon can be easilly isntalled via Helm

```bash
helm repo add isovalent <cilium_enterprise_repo>
helm repo update

helm upgrade --install tetragon isovalent/tetragon --version 1.15.1 \
--namespace tetragon --create-namespace --set tetragon.prometheus.serviceMonitor.enabled=true --set tetragonOperator.prometheus.serviceMonitor.enabled=true \
--set tetragon.prometheus.enabled=true --set tetragonOperator.prometheus.enabled=true
```

# Grafana

Once Tetragon is installed Prometheus should automatically picks up the Tetragon `servicemonitors` and start ingesting metrics.
For simplicity we have collected a few dashboards [here](https://github.com/datacenter/cilium-dc-design/tree/main/docs/tetragon/grafana/dashboards) you should be able to just apply them as they are. Just remember to apply them  to the namespace where `kube-prometheus-stack` is deployed.