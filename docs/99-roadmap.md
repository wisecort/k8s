# Roadmap

## Concluído

- [x] Proxmox / VMs
- [x] RKE2
- [x] 3 control planes
- [x] 3 workers
- [x] HAProxy
- [x] Keepalived
- [x] VIP 10.255.10.140
- [x] DNS interno
- [x] kubeconfig via VIP
- [x] Longhorn 1.12.1
- [x] 3 nós de storage Longhorn
- [x] data path /var/lib/longhorn
- [x] StorageClass padrão longhorn com 3 réplicas
- [x] StorageClass longhorn-prod
- [x] backup Longhorn em Cloudflare R2
- [x] Recurring Job diário do Longhorn
- [x] retenção de 60 backups
- [x] teste de backup e restore
- [x] Velero 1.18.2
- [x] bucket R2 separado para Velero
- [x] backup e restore de objetos Kubernetes
- [x] DR combinado Velero + Longhorn
- [x] cert-manager
- [x] PKI interna do Rancher
- [x] Rancher 2.15.1
- [x] Rancher via Traefik
- [x] Fleet operacional
- [x] kube-prometheus-stack 91.4.1
- [x] Grafana 13.2.2
- [x] rancher-monitoring-dashboards 110.0.0+up0.1.2
- [x] Rancher Monitoring disponível na UI
- [x] dashboards disponíveis no Grafana
- [x] acesso anônimo Viewer no Grafana
- [x] integração Grafana via proxy do Rancher

## Longhorn atual

- [x] StorageClass longhorn: 3 réplicas
- [x] StorageClass longhorn-prod: 1 réplica
- [x] backup recorrente diário às 02:00 UTC
- [x] retenção de 60 backups
- [x] Cloudflare R2
- [ ] monitoramento de falhas
- [ ] teste periódico de restore
- [ ] backup periódico do sistema Longhorn
- [ ] revisar independência física do storage

## Velero

- [x] Velero instalado
- [x] plugin AWS/S3
- [x] bucket R2 separado
- [x] BackupStorageLocation
- [x] backup de objetos
- [x] restore de objetos
- [x] DR combinado
- [ ] acompanhar estabilidade da integração R2/plugin
- [ ] validar Schedule automático
- [ ] monitoramento de falhas

## Rancher

- [x] Rancher 2.15.1
- [x] hostname rancher.k8s.internal
- [x] TLS interno via cert-manager
- [x] Ingress via Traefik
- [x] Fleet operacional
- [x] endpoint validado via HTTPS
- [x] Rancher Monitoring instalado
- [x] dashboards disponíveis no Grafana
- [x] integração Grafana via proxy
- [ ] instalar CA interna nas estações administrativas
- [ ] configurar alertas
- [ ] avaliar escala para 3 réplicas após estabilização do storage/I/O

## Ingress

- [x] Traefik
- [x] publicação interna do Rancher
- [ ] aplicação de teste

## Segurança

- [ ] NetworkPolicy
- [ ] ResourceQuota
- [ ] LimitRange
- [ ] RBAC
- [ ] Pod Security Standards
- [ ] isolamento por namespace

## Observabilidade

- [x] Rancher Monitoring runtime
- [x] Prometheus
- [x] Grafana
- [x] Alertmanager
- [x] node-exporter
- [x] kube-state-metrics
- [x] dashboards no Grafana
- [x] consultas Prometheus nos dashboards
- [ ] alertas
- [ ] Loki

## GitOps

- [ ] Argo CD
- [ ] estrutura declarativa de aplicações

## Disaster Recovery

- [x] restore de volume a partir do R2
- [x] restore de objetos via Velero
- [x] DR combinado
- [ ] perda de worker
- [ ] perda de host Proxmox
- [ ] perda de control plane
- [ ] recuperação completa
- [ ] RPO/RTO formal
- [ ] teste periódico
- [ ] automação do runbook
