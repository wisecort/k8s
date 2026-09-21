# Arquitetura de Backup e Disaster Recovery

## Objetivo

Separar as responsabilidades:

~~~text
Kubernetes Objects
        ↓
      Velero
        ↓
Cloudflare R2 / velero-backups-qg

PVC Data
        ↓
     Longhorn
        ↓
Cloudflare R2 / longhorn-backups-qg
~~~

São utilizados buckets R2 separados e credenciais independentes.

## Responsabilidades

### Velero

Protege objetos Kubernetes conforme o escopo do backup: namespaces, Deployments, StatefulSets, DaemonSets, Services, Ingresses, ConfigMaps, Secrets, ServiceAccounts, RBAC, Jobs, CronJobs, CRs/CRDs quando incluídos e metadados de PVC/PV.

Nesta arquitetura, Velero não é o backup primário do conteúdo dos PVCs.

### Longhorn

Protege os dados persistentes dos volumes e fornece snapshots, backups externos e restore.

PVCs que precisam da política automática utilizam:

~~~yaml
storageClassName: longhorn-prod
~~~

## Buckets

| Componente | Bucket |
|---|---|
| Velero | velero-backups-qg |
| Longhorn | longhorn-backups-qg |

Os buckets são privados.

**Nunca armazenar Access Key, Secret Key ou tokens no Git.**

## Política Longhorn atual

~~~text
StorageClass: longhorn-prod
Réplicas atuais: 1
Recurring Job: backup-daily-r2
Horário: 02:00 UTC
Retenção: 60 backups
Concorrência: 1
Backup Target: default
Backend: Cloudflare R2
~~~

A escolha de 1 réplica foi feita para reduzir a amplificação de I/O no ambiente atual, pois os workers estão no mesmo host físico Proxmox. Isso reduz a redundância local do volume e deve ser revisado antes de workloads críticos.

## Política Velero atual

~~~text
Schedule: backup-daily-r2
Horário: 04:00 UTC
Retenção: 30 dias
Bucket: velero-backups-qg
Snapshots de volume: desabilitados
~~~

## Fluxo de DR

~~~text
                DESASTRE
                    |
          +---------+---------+
          |                   |
          v                   v
      Velero R2          Longhorn R2
          |                   |
          v                   v
   Objetos Kubernetes     Volume/Data
          |                   |
          +---------+---------+
                    |
                    v
                Aplicação
                    |
                    v
              Dados íntegros
~~~

Ordem operacional:

1. confirmar backups;
2. restaurar objetos via Velero;
3. identificar PVCs que precisam de dados;
4. evitar usar PVC vazio como fonte dos dados;
5. restaurar volume via Longhorn;
6. associar o PVC ao workload;
7. iniciar aplicação;
8. validar dados;
9. validar checksum quando aplicável.

## Validação

### Velero

Backup e restore de objetos Kubernetes após perda controlada de namespace.

### Longhorn

Snapshot, Full Backup, envio para R2, restore por fromBackup e recuperação dos arquivos.

### DR combinado

Perda controlada de namespace, aplicação, PVC e volume Longhorn. Objetos recuperados por Velero e dados por Longhorn.

SHA-256 validado:

~~~text
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
~~~

## Limitação física

Os três workers RKE2 estão hospedados no mesmo Proxmox pve2 e os discos de root e Longhorn utilizam o mesmo backend físico NVME.

Logo, réplicas Longhorn em workers diferentes não garantem independência contra perda do host físico.

Para HA de storage real, as réplicas precisam estar em nós com independência física suficiente para o cenário de falha considerado.

## RPO/RTO

Os valores formais de RPO/RTO ainda não foram definidos.

Política atual:

- Longhorn: diário às 02:00 UTC, retenção 60;
- Velero: diário às 04:00 UTC, retenção 30 dias.

## Próximos testes

- [ ] monitoramento de falha de backup
- [ ] restore periódico automatizado
- [ ] perda de worker
- [ ] perda de host Proxmox
- [ ] perda de control plane
- [ ] recuperação completa do cluster
- [ ] definição formal de RPO/RTO
- [ ] automação do runbook
