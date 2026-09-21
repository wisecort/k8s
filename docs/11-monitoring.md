# Rancher Monitoring

## Estado atual

- Rancher: 2.15.1
- RKE2/Kubernetes: v1.36.4+rke2r1
- `kube-prometheus-stack`: 91.4.1
- Prometheus app: v0.94.0
- Prometheus: v3.14.0
- Grafana: 13.2.2
- `rancher-monitoring-dashboards`: 110.0.0+up0.1.2
- Namespace: `cattle-monitoring-system`
- Namespace dos dashboards: `cattle-dashboards`
- Runtime de monitoramento: operacional
- Integração com Rancher: operacional
- Grafana: operacional
- Dashboards: disponíveis no Grafana
- Acesso anônimo: habilitado como `Viewer`
- Alertmanager: operacional
- Discord: integração operacional e validada com alerta real
- Teste de alerta `KubePodCrashLooping`: validado de ponta a ponta
- Prometheus: retenção de 15 dias
- Prometheus: PVC de 10Gi em `longhorn-prod`
- Prometheus: Longhorn com 1 réplica
- Prometheus: requests 500m CPU / 1Gi RAM
- Prometheus: limits 2 CPU / 2Gi RAM
- Prometheus: `/-/ready` validado
- Prometheus TSDB: 247.790 séries observadas após a implantação da persistência
- Estado pendente no encerramento do teste: alerta ainda estava `firing` no Prometheus/Alertmanager após a exclusão do Pod; a resolução ainda precisa ser confirmada

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
       |                       |
       +-- kube-state-metrics  +-- AlertmanagerConfig
       |                       |
       +-- node-exporter       +-- Discord
~~~

## Instalação do runtime

A configuração declarativa atual está em:

~~~text
platform/monitoring/kube-prometheus-stack-values.yaml
~~~

Os valores incluem retenção, recursos, persistência do Prometheus e as opções necessárias para o ambiente RKE2. Segredos não são armazenados no arquivo.

~~~bash
helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace cattle-monitoring-system \
  --version 91.4.1 \
  --reset-values \
  -f platform/monitoring/kube-prometheus-stack-values.yaml \
  --wait \
  --timeout 10m
~~~

Configuração efetiva do Prometheus:

~~~yaml
prometheus:
  prometheusSpec:
    retention: 15d
    podMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelectorNilUsesHelmValues: false
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: "2"
        memory: 2Gi
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn-prod
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
~~~

O PVC foi criado com 10Gi em `longhorn-prod`. O volume Longhorn ficou `attached` e `healthy`, com uma única réplica, reduzindo a multiplicação de I/O no storage físico do Proxmox.

Validações realizadas:

~~~text
Prometheus Pod:       2/2 Running
PVC:                  Bound
PVC capacity:         10Gi
StorageClass:         longhorn-prod
Longhorn state:       attached
Longhorn robustness:  healthy
Longhorn replicas:    1
Prometheus /-/ready:  Prometheus Server is Ready.
Retention:            15d
TSDB corruption:      0
~~~

O endpoint de runtime também confirmou:

~~~json
{
  "reloadConfigSuccess": true,
  "corruptionCount": 0,
  "storageRetention": "15d"
}
~~~

O TSDB apresentou aproximadamente 247.790 séries no momento da validação inicial da persistência. O consumo deve ser acompanhado antes de aumentar ou reduzir a retenção.

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

Essas respostas estão relacionadas a funcionalidades de usuário/equipe não disponíveis para a identidade anônima e não impedem a visualização dos dashboards.

Os logs também confirmaram consultas Prometheus bem-sucedidas:

~~~text
endpoint=queryData
dsName=Prometheus
status=ok
~~~

Portanto, o estado atual é considerado operacional para visualização e monitoramento.

## Alertas e Discord

### Configuração

O Alertmanager utiliza uma `AlertmanagerConfig` chamada `discord` no namespace `cattle-monitoring-system`.

A configuração lógica é:

~~~yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: discord
  namespace: cattle-monitoring-system
spec:
  route:
    groupBy:
      - namespace
      - alertname
    groupInterval: 5m
    groupWait: 30s
    receiver: discord
    repeatInterval: 12h
  receivers:
    - name: discord
      discordConfigs:
        - apiURL:
            name: discord-webhook
            key: webhook-url
          sendResolved: true
~~~

O webhook fica armazenado no Secret `discord-webhook`; o valor do webhook não deve ser armazenado no Git nem documentado em texto.

O Alertmanager foi configurado para utilizar a `AlertmanagerConfig` como configuração principal. A configuração efetiva passou a utilizar:

~~~text
receiver: cattle-monitoring-system/discord/discord
discord_configs:
- send_resolved: true
~~~

Validação da configuração:

~~~bash
amtool check-config /etc/alertmanager/config_out/alertmanager.env.yaml
~~~

Resultado validado: `SUCCESS`.

### Teste sintético

Foi enviado um alerta sintético chamado `DiscordTest2` diretamente à API do Alertmanager.

O alerta apareceu no Alertmanager como `active` e foi recebido no Discord.

Esse teste comprovou a comunicação Alertmanager → Discord.

### Teste real de Kubernetes

Foi criado um Pod propositalmente com falha:

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: alert-test-crashloop
  namespace: default
spec:
  containers:
    - name: crash
      image: busybox:1.37
      command:
        - /bin/sh
        - -c
        - |
          echo "TESTE: pod propositalmente com problema"
          exit 1
~~~

O Pod entrou em falha e apresentou reinicializações.

O `kube-state-metrics` expôs `CrashLoopBackOff`, e a regra oficial `KubePodCrashLooping` entrou em `firing` após a duração configurada de 15 minutos.

O alerta chegou ao Discord, comprovando o fluxo real:

~~~text
Pod com problema
      |
      v
kube-state-metrics
      |
      v
Prometheus
      |
      v
KubePodCrashLooping
      |
      v
Alertmanager
      |
      v
AlertmanagerConfig
      |
      v
Discord
~~~

### Encerramento do teste

O Pod foi removido:

~~~bash
kubectl delete pod alert-test-crashloop
~~~

Após a exclusão, a consulta direta da métrica retornou `result: []`.

No último estado observado, o Prometheus ainda mostrava o alerta `KubePodCrashLooping` como `firing`, e o Alertmanager também o mostrava como `active`.

Portanto, a resolução do alerta após a remoção do Pod ainda precisa ser confirmada.

Comandos para retomar a validação:

~~~bash
curl -s http://127.0.0.1:9090/api/v1/alerts \
  | jq '.data.alerts[] |
  select(.labels.alertname=="KubePodCrashLooping") |
  {state:.state, labels:.labels, activeAt:.activeAt}'
~~~

E:

~~~bash
kubectl -n cattle-monitoring-system exec \
  alertmanager-kube-prometheus-stack-alertmanager-0 \
  -- amtool \
  --alertmanager.url=http://127.0.0.1:9093 \
  alert query alertname=KubePodCrashLooping
~~~

O objetivo final do teste é receber no Discord:

~~~text
[RESOLVED] KubePodCrashLooping
~~~

## Estado validado

~~~text
Prometheus                  OK
Grafana                     OK
Alertmanager                OK
Node Exporter               OK
kube-state-metrics          OK
Rancher Monitoring          OK
Dashboard ConfigMaps        OK
Dashboards Grafana          OK
Prometheus queries          OK
AlertmanagerConfig          OK
Discord notification        OK
KubePodCrashLooping         OK / firing test validated
Resolution after deletion   PENDENTE DE CONFIRMAÇÃO
Prometheus persistence      OK
Prometheus retention        15d
Prometheus PVC              10Gi
Prometheus Longhorn         1 replica / healthy
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

O webhook do Discord foi exposto durante a fase de testes. Antes de considerar essa integração definitiva, o webhook deve ser rotacionado/revogado e mantido exclusivamente no Secret `discord-webhook`, sem valor em Git ou nos valores Helm.

Como a configuração do Prometheus foi consolidada em arquivo versionado, alterações futuras devem ser feitas em `platform/monitoring/kube-prometheus-stack-values.yaml` e aplicadas com Helm, evitando alterações manuais no StatefulSet.

## Próximos passos

- [x] instalar runtime de monitoramento
- [x] instalar dashboards do Rancher
- [x] integrar Monitoring à UI do Rancher
- [x] corrigir `clusterId=local` do proxy
- [x] dashboards disponíveis no Grafana
- [x] Prometheus alimentando dashboards
- [x] acesso anônimo Viewer
- [x] configurar Alertmanager
- [x] integrar Alertmanager com Discord
- [x] validar alerta sintético no Discord
- [x] validar alerta real `KubePodCrashLooping`
- [x] configurar retenção de 15 dias
- [x] configurar PVC persistente de 10Gi
- [x] configurar recursos CPU/memória do Prometheus
- [x] manter Prometheus com 1 réplica Longhorn
- [ ] confirmar `RESOLVED` após remoção do Pod de teste
- [ ] rotacionar webhook do Discord e remover valor dos Helm values históricos
- [ ] acompanhar consumo do PVC e TSDB
- [ ] avaliar Loki
