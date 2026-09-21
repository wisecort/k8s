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
- [x] Longhorn
- [x] 3 réplicas Longhorn
- [x] teste de PVC
- [x] Backup Longhorn em Cloudflare R2
- [x] teste de restore a partir do R2
- [x] validação dos dados restaurados
- [x] StorageClass `longhorn-prod`
- [x] associação automática de novos volumes ao grupo de backup
- [x] Recurring Job diário do Longhorn
- [x] retenção de 60 backups
- [x] execução manual do Recurring Job
- [x] validação de backup recorrente no R2
- [x] instalação do Velero 1.18.2
- [x] criação do bucket R2 separado para Velero
- [x] BackupStorageLocation do Velero validado como Available
- [x] backup de objetos Kubernetes
- [x] perda controlada de namespace de teste
- [x] restore de objetos Kubernetes via Velero
- [x] aplicação de teste restaurada e Running

## Próximos passos

### 1. Backup Longhorn

- [x] destino S3-compatible
- [x] teste de backup Full
- [x] teste de restore
- [x] política de backup
- [x] retenção de 60 backups
- [x] Recurring Backup Job diário às 02:00
- [ ] monitoramento de falhas
- [ ] teste periódico de restore
- [ ] backup periódico do sistema Longhorn

### 2. Velero

- [x] Velero instalado
- [x] plugin AWS/S3 instalado
- [x] bucket R2 separado
- [x] BackupStorageLocation
- [x] validação de acesso ao R2
- [x] backup de objetos Kubernetes
- [x] restore de objetos Kubernetes
- [x] teste de perda de namespace
- [ ] acompanhar estabilidade da integração `x-amz-tagging` / R2
- [ ] política de retenção de 30 dias
- [ ] Schedule diário às 04:00
- [ ] validar backup automático do Schedule
- [ ] monitoramento de falhas
- [ ] teste de Disaster Recovery completo com PVC

### 3. Ingress

- [ ] HAProxy 80/443
- [ ] Traefik
- [ ] publicação de aplicação de teste

### 4. TLS

- [ ] cert-manager
- [ ] Let's Encrypt
- [ ] DNS-01

### 5. Segurança

- [ ] NetworkPolicy
- [ ] ResourceQuota
- [ ] LimitRange
- [ ] RBAC
- [ ] Pod Security Standards
- [ ] isolamento por namespace

### 6. Observabilidade

- [ ] Prometheus
- [ ] Grafana
- [ ] Alertmanager
- [ ] Loki

### 7. GitOps

- [ ] Argo CD
- [ ] estrutura declarativa de aplicações

### 8. Disaster Recovery

- [x] restore de volume a partir do R2
- [x] restore de objetos Kubernetes via Velero
- [ ] DR combinado: objetos Velero + dados Longhorn
- [ ] perda de worker
- [ ] perda de control plane
- [ ] recuperação completa
- [ ] teste periódico de DR
