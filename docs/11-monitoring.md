# Rancher Monitoring

## Estado atual

- Rancher: 2.15.1
- RKE2/Kubernetes: v1.36.4+rke2r1
- `kube-prometheus-stack`: 91.4.1
- Prometheus app: v0.94.0
- Grafana: 13.2.2
- `rancher-monitoring-dashboards`: 110.0.0+up0.1.2
- Namespace: `cattle-monitoring-system`
- Namespace dos dashboards: `cattle-dashboards`
- Runtime de monitoramento: operacional
- Integração com Rancher: operacional
- Grafana: operacional
- Dashboards: disponíveis no Grafana
- Acesso anônimo: habilitado como `Viewer`

## Arquitetura

A partir do Rancher 2.15, o monitoramento utiliza uma arquitetura desacoplada: o `kube-prometheus-stack` fornece o runtime de observabilidade e o `rancher-monitoring-dashboards` fornece os artefatos de dashboards e a integração com a UI do Rancher.

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
                              Grafana

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

O `rancherMonitoring.enabled=false` foi utilizado porque a instalação inicial apresentou rejeição de um `ServiceMonitor` com `metricRelabelings: null`.

## Integração com o Rancher

Como a instalação do `rancher-monitoring-dashboards` foi feita diretamente via Helm, foi necessário informar o cluster local:

~~~bash
helm upgrade rancher-monitoring-dashboards \
  rancher-charts/rancher-monitoring-dashboards \
  --namespace cattle-monitoring-system \
  --version 110.0.0+up0.1.2 \
  --reuse-values \
  --set global.cattle.clusterId=local \
  --set global.cattle.clusterName=local \
  --wait \
  --timeout 10m
~~~

Sem esse valor, o `appSubUrl` do proxy era gerado com `/k8s/clusters//`, causando `Page not found` no acesso pelo Rancher.

Após o ajuste, o proxy passou a utilizar:

~~~text
/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy
~~~

O Service `rancher-monitoring-grafana` é um proxy Nginx que encaminha para o Grafana real:

~~~text
rancher-monitoring-grafana:80
        |
        v
monitoring-proxy:8080
        |
        v
kube-prometheus-stack-grafana:80
        |
        v
Grafana:3000
~~~

## Acesso anônimo ao Grafana

Foi habilitado acesso anônimo somente como Viewer.

Configuração efetiva:

~~~ini
[auth.anonymous]
enabled = true
org_role = Viewer
~~~

O valor foi aplicado ao `grafana.ini` usando um arquivo de valores adicional:

~~~yaml
grafana:
  grafana.ini:
    auth.anonymous:
      enabled: true
      org_role: Viewer
~~~

Validação:

~~~bash
kubectl -n cattle-monitoring-system get cm kube-prometheus-stack-grafana \
  -o jsonpath='{.data.grafana\\.ini}' | grep -A3 '\\[auth.anonymous\\]'
~~~

Resultado validado:

~~~text
[auth.anonymous]
enabled = true
org_role = Viewer
~~~

## Comportamento de autenticação

O acesso anônimo permite abrir os dashboards e consultar métricas como Viewer.

Durante a validação, o Grafana registrou `401` em algumas chamadas de APIs específicas de usuário:

~~~text
/api/user/stars
/api/user/teams
~~~

Também foi observado `403` em:

~~~text
/api/teams/search
~~~

Essas respostas estão relacionadas a funcionalidades de usuário/equipe não disponíveis para a identidade anônima e **não impedem a visualização dos dashboards**.

Os logs também confirmaram consultas Prometheus bem-sucedidas:

~~~text
endpoint=queryData
dsName=Prometheus
status=ok
~~~

Portanto, o estado atual é considerado operacional para visualização e monitoramento.

## Estado validado

~~~text
Prometheus          OK
Grafana             OK
Alertmanager        OK
Node Exporter       OK
kube-state-metrics  OK
Rancher Monitoring  OK
Dashboard ConfigMaps OK
Dashboards Grafana  OK
Prometheus queries  OK
~~~

Pods validados em `cattle-monitoring-system`:

~~~text
alertmanager-kube-prometheus-stack-alertmanager-0
kube-prometheus-stack-grafana
kube-prometheus-stack-kube-state-metrics
kube-prometheus-stack-operator
kube-prometheus-stack-prometheus-node-exporter-*
prometheus-kube-prometheus-stack-prometheus-0
rancher-monitoring-dashboards-monitoring-proxy
~~~

## Dashboards

Os artefatos de dashboards são criados como ConfigMaps no namespace `cattle-dashboards`, usando o label:

~~~text
grafana_dashboard=1
~~~

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

Comando:

~~~bash
kubectl get cm -n cattle-dashboards -l grafana_dashboard=1
~~~

## Operação

~~~bash
kubectl get pods -n cattle-monitoring-system
kubectl get svc -n cattle-monitoring-system
kubectl get cm -n cattle-dashboards -l grafana_dashboard=1
helm list -n cattle-monitoring-system
~~~

Para validar o Grafana diretamente:

~~~bash
kubectl -n cattle-monitoring-system port-forward svc/kube-prometheus-stack-grafana 3000:80
~~~

Health check:

~~~bash
curl -s http://localhost:3000/api/health
~~~

Resposta validada:

~~~json
{
  "database": "ok",
  "version": "13.2.2"
}
~~~

## Observação operacional

O Grafana está operacional e os dashboards estão disponíveis. O `Unauthorized`/HTTP 401 observado em algumas chamadas de usuário/equipe é uma limitação das APIs específicas para acesso anônimo e não bloqueia os dashboards.

Caso seja necessário eliminar também essas respostas para funcionalidades administrativas, deve-se avaliar posteriormente autenticação integrada/RBAC em vez de ampliar as permissões da identidade anônima.

## Próximos passos

- [x] instalar runtime de monitoramento
- [x] instalar dashboards do Rancher
- [x] integrar Monitoring à UI do Rancher
- [x] corrigir `clusterId=local` do proxy
- [x] dashboards disponíveis no Grafana
- [x] Prometheus alimentando dashboards
- [x] acesso anônimo Viewer
- [ ] configurar alertas
- [ ] avaliar Loki
- [ ] revisar retenção de métricas
- [ ] revisar recursos CPU/memória do stack
