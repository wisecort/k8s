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

## Próximos passos

### 1. Backup Longhorn

- [x] destino S3-compatible
- [x] teste de backup Full
- [x] teste de restore
- [ ] política de backup
- [ ] retenção
- [ ] Recurring Backup Jobs
- [ ] monitoramento de falhas
- [ ] teste periódico de restore
- [ ] backup periódico do sistema Longhorn

### 2. Ingress
- [ ] HAProxy 80/443
- [ ] Traefik
- [ ] publicação de aplicação de teste

### 3. TLS
- [ ] cert-manager
- [ ] Let's Encrypt
- [ ] DNS-01

### 4. Segurança
- [ ] NetworkPolicy
- [ ] ResourceQuota
- [ ] LimitRange
- [ ] RBAC
- [ ] Pod Security Standards
- [ ] isolamento por namespace

### 5. Observabilidade
- [ ] Prometheus
- [ ] Grafana
- [ ] Alertmanager
- [ ] Loki

### 6. GitOps
- [ ] Argo CD
- [ ] estrutura declarativa de aplicações

### 7. Disaster Recovery
- [x] restore de volume a partir do R2
- [ ] perda de worker
- [ ] perda de control plane
- [ ] recuperação completa
