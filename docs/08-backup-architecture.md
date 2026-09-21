# Arquitetura de Backup e Disaster Recovery

## Objetivo

Definir a arquitetura de proteção e recuperação do cluster RKE2 separando as responsabilidades de Velero, Longhorn e Cloudflare R2.

## Visão geral

```
                         RKE2
                          |
              +-----------+-----------+
              |                       |
         Kubernetes                Longhorn
          Objects                  Volumes
              |                       |
           Velero                  Backup
              |                       |
              +-----------+-----------+
                          |
                    Cloudflare R2
                   /              \
          velero-backups-qg   longhorn-backups-qg
```

São utilizados dois buckets R2 separados, com credenciais independentes.

## Responsabilidades

### Velero

Protege o estado e os objetos Kubernetes:

- Namespace;
- Deployment;
- StatefulSet;
- DaemonSet;
- Service;
- Ingress;
- ConfigMap;
- Secret;
- ServiceAccount;
- RBAC;
- Job;
- CronJob;
- CRs/CRDs quando incluídos;
- PersistentVolumeClaim;
- metadados de PersistentVolume.

Nesta arquitetura, Velero não é o backup primário do conteúdo dos PVCs.

### Longhorn

Protege os dados persistentes dos volumes:

- filesystem;
- arquivos;
- snapshots;
- réplicas;
- backups externos;
- restauração de volumes.

PVCs de produção devem utilizar `longhorn-prod`.

### Cloudflare R2

| Componente | Bucket |
|---|---|
| Velero | `velero-backups-qg` |
| Longhorn | `longhorn-backups-qg` |

Os buckets são privados.

**Nunca armazenar Access Key, Secret Key ou tokens no Git.**

## Separação de responsabilidades

```
Velero  -> "o que existe no Kubernetes"
Longhorn -> "os dados que existem no volume"
R2      -> armazenamento externo dos dois conjuntos
```

Um restore Velero isolado não deve ser considerado recuperação do conteúdo de um volume Longhorn.

## Fluxo de backup

### Objetos Kubernetes

```
Deployment / Service / ConfigMap / Secret / PVC
                         |
                       Velero
                         |
                         v
                  velero-backups-qg
```

### Dados persistentes

```
PVC
 |
Longhorn Volume
 |
Snapshot / Backup
 |
v
longhorn-backups-qg
```

## Fluxo de Disaster Recovery

```
                 DESASTRE
                    |
          +---------+---------+
          |                   |
          v                   v
      Velero R2          Longhorn R2
          |                   |
          v                   v
   Objetos K8s            Volume/Data
          |                   |
          +---------+---------+
                    |
                    v
               Aplicação
                    |
                    v
              Dados íntegros
```

Ordem operacional:

1. confirmar disponibilidade dos backups;
2. restaurar objetos via Velero;
3. não utilizar o PVC vazio provisionado automaticamente como fonte dos dados;
4. restaurar o volume via Longhorn;
5. associar o workload ao PVC restaurado;
6. iniciar a aplicação;
7. validar os dados;
8. validar integridade por checksum quando aplicável.

## Política

### Longhorn

- diário às 02:00;
- retenção de 60 backups;
- concorrência 1;
- grupo `default`;
- destino R2;
- StorageClass de produção `longhorn-prod`.

### Velero

- diário às 04:00;
- retenção de 30 dias;
- destino R2;
- objetos Kubernetes;
- snapshots de volume desabilitados para esta arquitetura;
- dados dos PVCs protegidos pelo Longhorn.

## Validação realizada

A arquitetura foi validada em três níveis:

### Velero

Backup e restore de objetos Kubernetes após perda controlada de namespace.

### Longhorn

Backup e restore de volume após perda do volume original, com validação do conteúdo por SHA-256.

### DR combinado

Foi simulada a perda de namespace, aplicação, PVC e volume Longhorn. Os objetos foram recuperados pelo Velero e os dados pelo Longhorn.

SHA-256 original e restaurado:

```
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
```

## Critério de sucesso

- [x] backup Velero disponível;
- [x] backup Longhorn disponível;
- [x] namespace restaurado;
- [x] objetos restaurados;
- [x] PVC restaurado;
- [x] volume restaurado;
- [x] aplicação iniciada;
- [x] dados originais recuperados;
- [x] integridade validada.

## Próximas evoluções

- monitoramento e alertas;
- testes periódicos de restore;
- perda de worker;
- perda de control plane;
- recuperação completa do cluster;
- automação do runbook de DR.
