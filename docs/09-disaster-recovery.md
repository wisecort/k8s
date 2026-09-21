# Disaster Recovery — Restore Combinado

## Objetivo

Recuperar uma aplicação Kubernetes após perda dos objetos Kubernetes e do volume persistente, usando Velero para objetos e Longhorn para dados.

## Cenário validado

Namespace:

```
dr-test
```

Aplicação:

```
Deployment: dr-test
Image: nginx:1.29-alpine
Replicas: 1
```

PVC:

```
dr-test-pvc
StorageClass: longhorn-prod
AccessMode: ReadWriteOnce
Size: 1Gi
```

Dados:

```
/data/dr-test.txt
/data/data-criacao.txt
/data/hostname.txt
/data/teste.bin
/data/teste.bin.sha256
```

SHA-256 registrado antes do desastre:

```
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
```

## Backup Velero

Backup:

```
dr-test-combined
```

Resultado:

```
Completed
32/32 itens
0 erros
0 warnings
Velero-Native Snapshot PVs: false
CSI Snapshots: none
Pod Volume Backups: none
```

O backup contém Namespace, Deployment, ReplicaSet, Service, ConfigMaps, Secret, PVC, PV metadata e ServiceAccount.

## Backup Longhorn

Backup:

```
backup-df6691ad8e4e450f
```

Volume original:

```
pvc-4d7f4ce2-15ea-4cff-9b26-51797719fa2f
```

Resultado:

```
Completed
Progress: 100
Size: 77594624
Compression: lz4
Backup Target: default
```

## Simulação do desastre

Foi executado:

```bash
kubectl delete namespace dr-test
```

O namespace, PVC e volume original foram removidos. O backup Longhorn permaneceu disponível no R2.

## Etapa 1 — Restore Velero

```bash
velero restore create dr-test-combined-restore \
  --from-backup dr-test-combined \
  --restore-volumes=false \
  --wait
```

Resultado:

```
Completed
13/13 itens restaurados
0 erros
1 warning
```

A única warning foi referente a `kube-root-ca.crt`, recurso gerenciado pelo Kubernetes.

O Velero recriou o PVC, mas esse PVC apontava para um novo volume vazio.

## Etapa 2 — Substituição do volume vazio

O Deployment foi escalado para zero e o PVC vazio foi removido. Como `longhorn-prod` usa `reclaimPolicy: Delete`, o volume vazio também foi eliminado.

## Etapa 3 — Restore Longhorn

Foi utilizada uma StorageClass temporária com `fromBackup`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-dr-combined
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: "s3://longhorn-backups-qg@auto/backupstore?backup=backup-df6691ad8e4e450f&volume=pvc-4d7f4ce2-15ea-4cff-9b26-51797719fa2f"
  fsType: "ext4"
  dataLocality: "disabled"
  unmapMarkSnapChainRemoved: "ignored"
  disableRevisionCounter: "true"
  dataEngine: "v1"
  backupTargetName: "default"
```

O PVC foi recriado com o nome esperado pela aplicação e ficou associado ao volume restaurado.

O volume restaurado foi validado com 3 réplicas e estado saudável.

## Etapa 4 — Validação da aplicação

O Deployment foi iniciado novamente e o arquivo foi validado:

```bash
kubectl -n dr-test exec deploy/dr-test -- sha256sum /data/teste.bin
```

Resultado:

```
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
```

O SHA-256 é idêntico ao valor registrado antes da destruição.

## Resultado

```
             DESASTRE
                |
      +---------+---------+
      |                   |
      v                   v
  Velero R2          Longhorn R2
      |                   |
      v                   v
 Objetos K8s          Volume/Data
      |                   |
      +---------+---------+
                |
                v
              PVC
                |
                v
           Deployment
                |
                v
            Aplicação
                |
                v
             Dados
                |
                v
           SHA-256 OK
```

## Resultado final

| Camada | Resultado |
|---|---|
| Velero backup | OK |
| Velero restore | OK |
| Namespace | restaurado |
| Deployment | restaurado |
| Service | restaurado |
| ConfigMap | restaurado |
| Secret | restaurado |
| PVC metadata | restaurado |
| Longhorn backup | OK |
| Longhorn restore | OK |
| Réplicas Longhorn | 3 |
| Arquivos | restaurados |
| SHA-256 | idêntico |

## Runbook operacional

1. confirmar backup Velero;
2. confirmar backup Longhorn;
3. restaurar objetos com Velero sem volumes;
4. identificar PVCs que precisam de dados;
5. remover ou evitar PVC vazio criado automaticamente;
6. restaurar cada volume pelo Longhorn;
7. associar o PVC restaurado ao workload;
8. iniciar a aplicação;
9. validar saúde;
10. validar dados críticos;
11. registrar evidências do DR.

## Limitações ainda não testadas

- perda de worker;
- perda de control plane;
- perda simultânea de múltiplos workers;
- recuperação completa do cluster;
- StatefulSets complexos;
- cenários RWX;
- automação completa do runbook.
