# Kubernetes Platform

Documentação da plataforma Kubernetes baseada em RKE2 e Proxmox.

## URLs

### Administração

- **Rancher:** https://rancher.k8s.internal
- **Grafana:** https://rancher.k8s.internal/api/v1/namespaces/cattle-monitoring-system/services/http:rancher-monitoring-grafana:80/proxy/
- **Kubernetes API:** https://k8s-api.k8s.internal:6443

### Domínio público

- **Domínio:** https://matheus.app.br

> O Rancher e o Kubernetes API são endpoints internos da plataforma. O Grafana é acessado através do proxy do Rancher.

## Documentação

- [Arquitetura](docs/00-architecture.md)
- [Inventário](inventory/nodes.md)
- [Rede e VIP](docs/02-networking.md)
- [RKE2](docs/03-rke2.md)
- [HAProxy e Keepalived](docs/04-ha.md)
- [Longhorn](docs/05-longhorn.md)
- [DNS](docs/06-dns.md)
- [Velero](docs/07-velero.md)
- [Arquitetura de Backup e DR](docs/08-backup-architecture.md)
- [Disaster Recovery](docs/09-disaster-recovery.md)
- [Rancher](docs/10-rancher.md)
- [Monitoring](docs/11-monitoring.md)
- [Roadmap](docs/99-roadmap.md)

## Princípio

A documentação descreve o ambiente real e deve ser atualizada junto com mudanças de infraestrutura.

Credenciais, tokens, kubeconfigs e Secrets não devem ser versionados.
