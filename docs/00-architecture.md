# Arquitetura

## Visão geral

```
Internet
   |
Public IP / NAT
   |
HAProxy + Keepalived
   |
Kubernetes API / Traefik
   |
RKE2
   |
+-------------------------------+
| CP01 CP02 CP03               |
|                               |
| W01 W02 W03                  |
+-------------------------------+
             |
          Longhorn
       3 replicas/volume
```

## Componentes

| Componente | Função |
|---|---|
| Proxmox | Virtualização |
| RKE2 | Kubernetes |
| HAProxy | Balanceamento |
| Keepalived | VIP e failover dos LBs |
| PowerDNS | DNS authoritative |
| Recursive DNS | Resolução dos nós |
| Longhorn | Storage persistente distribuído |
| Traefik | Ingress, instalado pelo RKE2 |

## Princípios de alta disponibilidade

O control plane possui três nós. O acesso ao Kubernetes API e ao supervisor RKE2 utiliza o VIP 10.255.10.140.

Longhorn utiliza três workers e réplica padrão 3.
