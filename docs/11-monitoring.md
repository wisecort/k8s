# Rancher Monitoring

## Estado atual

- Rancher: 2.15.1
- RKE2/Kubernetes: v1.36.4+rke2r1
- `kube-prometheus-stack`: 91.4.1
- Prometheus app: v0.94.0
- `rancher-monitoring-dashboards`: 110.0.0+up0.1.2
- Namespace: `cattle-monitoring-system`
- Namespace dos dashboards: `cattle-dashboards`
- Runtime de monitoramento: operacional
- Integração com Rancher: operacional; Monitoring aparece na UI
- Grafana: operacional, porém os dashboards ainda não estão aparecendo na interface Grafana

## Arquitetura

A partir do Rancher 2.15, o monitoramento novo utiliza uma arquitetura desacoplada: o `kube-prometheus-stack` fornece o runtime de observabilidade e o `rancher-monitoring-dashboards` fornece os artefatos de dashboards e a integração com a UI do Rancher.

~~~text
                  Rancher 2.15.1
                       |
                       v
          rancher-monitoring-dashboards
                       |
             +---------+---------+
             |                   |
             v                   v
       Rancher UI           cattle-dashboards
                                 |
                                 v
                           Grafana dashboards

          kube-prometheus-stack
             |
       +-----+------+---------+
       |            |         |
       v            v         v
   Prometheus    Grafana   Alertmanager
       |
       +-- kube-state-metrics
       |
       +-- node-exporter
~~~

A documentação oficial do Rancher 2.15 descreve essa separação entre o runtime `kube-prometheus-stack` e o chart `rancher-monitoring-dashboards`. citeturn0search0

## Instalação do runtime

~~~bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace cattle-monitoring-system \
  --version 91.4.1 \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set kubeEtcd.enabled=false \
  --set kubeControllerManager.enabled=false \
  --set kubeScheduler.enabled=false \
  --set kubeProxy.enabled=false \
  --wait \
  --timeout 10m
~~~

O Prometheus foi instalado com os selectors de ServiceMonitor e PodMonitor abertos para permitir a integração com os monitores do cluster.

Os exporters de etcd, controller-manager, scheduler e kube-proxy foram deixados desabilitados. Essa configuração é compatível com o modelo documentado pelo Rancher para o novo chart de dashboards. citeturn0search0

## Rancher Monitoring Dashboards

~~~bash
helm upgrade --install rancher-monitoring-dashboards \
  rancher-charts/rancher-monitoring-dashboards \
  --namespace cattle-monitoring-system \
  --version 110.0.0+up0.1.2 \
  --set rancherMonitoring.enabled=false \
  --wait \
  --timeout 10m
~~~

O `rancherMonitoring.enabled=false` foi utilizado porque a instalação inicial apresentou rejeição de um `ServiceMonitor` com:

~~~text
spec.endpoints[0].metricRelabelings:
Invalid value: "null"
~~~

Com a integração específica de métricas do chart desabilitada, a instalação foi concluída normalmente e o Rancher passou a exibir a área Monitoring.

## Estado validado

~~~text
cattle-monitoring-system

alertmanager-kube-prometheus-stack-alertmanager-0                 2/2 Running
kube-prometheus-stack-grafana                                     3/3 Running
kube-prometheus-stack-kube-state-metrics                          1/1 Running
kube-prometheus-stack-operator                                    1/1 Running
kube-prometheus-stack-prometheus-node-exporter                     6/6 Running
prometheus-kube-prometheus-stack-prometheus-0                    2/2 Running
rancher-monitoring-dashboards-monitoring-proxy                    1/1 Running
~~~

O Rancher já apresenta o menu/área de Monitoring.

## Serviços de integração

O chart criou os serviços esperados pelo Rancher:

~~~text
rancher-monitoring-prometheus
rancher-monitoring-grafana
rancher-monitoring-alertmanager
~~~

Os endpoints validados foram:

~~~text
rancher-monitoring-prometheus     10.42.3.34:8090
rancher-monitoring-grafana        10.42.3.34:8080
rancher-monitoring-alertmanager   10.42.3.34:8093
~~~

Esses endpoints são fornecidos pelo proxy do chart para a integração da UI do Rancher.

## Dashboards

Os artefatos de dashboards foram criados no namespace `cattle-dashboards`.

Exemplos validados:

~~~text
rancher-backup-restore-dashboard
rancher-default-dashboards-cluster
rancher-default-dashboards-home
rancher-default-dashboards-k8s
rancher-default-dashboards-nodes
rancher-default-dashboards-performance-debugging
rancher-default-dashboards-pods
rancher-default-dashboards-workloads
rancher-fleet-dashboards
rancher-fluentbit-dashboard
rancher-fluentd-dashboard
~~~

Comando utilizado:

~~~bash
kubectl get cm -n cattle-dashboards -l grafana_dashboard=1
~~~

## Pendência atual

O runtime está saudável e os ConfigMaps dos dashboards existem, mas os dashboards ainda não aparecem no Grafana.

Estado atual:

~~~text
Prometheus       OK
Grafana          OK
Alertmanager     OK
Node Exporter    OK
kube-state-metrics OK
Rancher Monitoring UI OK
Dashboard ConfigMaps OK
Dashboards dentro do Grafana PENDENTE
~~~

A investigação deve continuar no sidecar de dashboards do Grafana e na forma como ele está observando o namespace `cattle-dashboards`.

Não considerar essa etapa concluída até os dashboards aparecerem no Grafana.

## Observação

A documentação oficial do Rancher 2.15 informa que o `rancher-monitoring-dashboards` fornece os dashboards e a integração com a UI, enquanto Prometheus, Grafana, Alertmanager e exporters são fornecidos separadamente pelo `kube-prometheus-stack`. citeturn0search0

## Operação

~~~bash
kubectl get pods -n cattle-monitoring-system
kubectl get svc -n cattle-monitoring-system
kubectl get cm -n cattle-dashboards -l grafana_dashboard=1
helm list -n cattle-monitoring-system
~~~
