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

## Próximos passos

### 1. Backup Longhorn
- [ ] destino S3-compatible
- [ ] política de backup
- [ ] retenção
- [ ] teste de restore

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
- [ ] perda de worker
- [ ] restore de volume
- [ ] perda de control plane
- [ ] recuperação completa
