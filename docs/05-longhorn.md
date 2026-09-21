# Longhorn

## Estado atual

- Longhorn: **1.12.1**
- Data engine: **v1**
- Nós de storage: rke2-worker01, rke2-worker02, rke2-worker03
- Control planes: scheduling do Longhorn desabilitado
- Data path: /var/lib/longhorn
- Backup externo: Cloudflare R2
- Backup target: default

> Importante: os três workers estão no mesmo host físico Proxmox pve2. As réplicas do Longhorn fornecem redundância entre VMs/nós Kubernetes, mas não independência contra falha do host físico ou do storage físico compartilhado.

## StorageClass padrão

A StorageClass longhorn é a StorageClass padrão criada/gerenciada pelo Longhorn.

Estado atual:

~~~text
name: longhorn
provisioner: driver.longhorn.io
default: true
numberOfReplicas: 3
fsType: ext4
dataEngine: v1
dataLocality: disabled
backupTargetName: default
~~~

A StorageClass padrão foi mantida separada da StorageClass de produção.

Não alterar manualmente parâmetros gerenciados pelo Longhorn para tentar forçar uma configuração diferente. Alterações globais devem ser feitas pela configuração do Helm/Longhorn.

## StorageClass de produção

Foi criada uma StorageClass específica:

longhorn-prod

Configuração atual:

~~~yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-prod
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
  dataLocality: "disabled"
  unmapMarkSnapChainRemoved: "ignored"
  disableRevisionCounter: "true"
  dataEngine: "v1"
  backupTargetName: "default"
  recurringJobSelector: '[{"name":"default","isGroup":true}]'
~~~

PVCs que precisam da política automática de backup devem usar:

~~~yaml
spec:
  storageClassName: longhorn-prod
~~~

### Motivo da 1 réplica

A configuração atual foi definida para reduzir a amplificação de I/O no ambiente, pois os três workers compartilham o mesmo storage físico do host Proxmox pve2.

Isso é uma decisão operacional do ambiente atual e não equivale a HA de storage físico. Com uma réplica, a disponibilidade do volume depende do nó que hospeda a réplica.

Antes de workloads críticos, avaliar storage físico independente entre hosts Proxmox e/ou retornar volumes críticos para 3 réplicas em nós fisicamente independentes.

## Cloudflare R2

Bucket:

~~~text
longhorn-backups-qg
~~~

Região: auto

Path:

~~~text
s3://longhorn-backups-qg@auto/backupstore
~~~

Secret:

~~~text
cloudflare-r2-longhorn
Namespace: longhorn-system
~~~

O endpoint S3-compatible é armazenado no Secret através de AWS_ENDPOINTS.

**Credenciais, Access Key, Secret Key e tokens nunca devem ser armazenados no Git.**

BackupTarget:

~~~yaml
apiVersion: longhorn.io/v1beta2
kind: BackupTarget
metadata:
  name: default
  namespace: longhorn-system
spec:
  backupTargetURL: s3://longhorn-backups-qg@auto/backupstore
  credentialSecret: cloudflare-r2-longhorn
  pollInterval: 5m
~~~

Verificar:

~~~bash
kubectl -n longhorn-system get backuptarget default
~~~

## Backup recorrente

Foi criada:

~~~yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: backup-daily-r2
  namespace: longhorn-system
spec:
  cron: "0 2 * * *"
  task: backup
  groups:
    - default
  retain: 60
  concurrency: 1
~~~

Política:

| Parâmetro | Valor |
|---|---|
| Frequência | diária |
| Horário | 02:00 UTC |
| Tarefa | backup |
| Grupo | default |
| Retenção | 60 backups |
| Concorrência | 1 |
| Destino | BackupTarget default |
| Backend | Cloudflare R2 |

A longhorn-prod seleciona o grupo default através de recurringJobSelector.

## Backup recorrente validado

Execução manual:

~~~text
Job: backup-daily-r2-manual-1789947433
Status: Complete
Completions: 1/1
Duration: 29s
~~~

Backup produzido:

~~~text
Backup: backup-40c3f6c4e3b9452d
State: Completed
Size: 67108864
Volume: pvc-d281dc1f-6b33-4889-a640-4b92fc87290b
~~~

O volume registrou:

~~~text
LAST-BACKUP: backup-40c3f6c4e3b9452d
LAST-BACKUP-AT: 2026-09-20T23:37:21Z
STATE: attached
ROBUSTNESS: healthy
~~~

## Teste manual de backup e restore

Foi criado um PVC de 1Gi com dados de teste, incluindo teste.txt, data-criacao.txt, hostname.txt e teste.bin de 10 MiB.

teste.txt:

~~~text
LONGHORN-R2-TEST
~~~

Fluxo validado:

~~~text
PVC
 ↓
Longhorn Volume
 ↓
Snapshot
 ↓
Full Backup
 ↓
Cloudflare R2
 ↓
StorageClass temporária com fromBackup
 ↓
PVC restaurado
 ↓
Pod
 ↓
Dados recuperados
~~~

Backup:

~~~text
backup-d1107a4efea04084
State: Completed
Progress: 100
Compression: lz4
~~~

O restore recuperou teste.txt e o arquivo binário de 10 MiB.

SHA-256 registrado no restore:

~~~text
15bf54eef18075760401db54095cd988c773750a32515114f1fa78ba47a53b3e
~~~

O SHA-256 original desse teste não foi preservado nesta documentação; portanto, não declarar igualdade byte-a-byte para esse teste.

## Restore

O mecanismo utilizado é parameters.fromBackup.

~~~yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-dr-restore
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: "s3://longhorn-backups-qg@auto/backupstore?backup=<backup>&volume=<volume>"
  fsType: "ext4"
  dataLocality: "disabled"
  unmapMarkSnapChainRemoved: "ignored"
  disableRevisionCounter: "true"
  dataEngine: "v1"
  backupTargetName: "default"
~~~

## Observação sobre snapshots

Foi tentada inicialmente a criação de snapshot/backup via CR de forma incorreta e o Longhorn retornou erro relacionado ao controle interno do snapshot.

O fluxo validado foi criar o snapshot pela UI do Longhorn e executar o backup a partir de um snapshot válido.

Para Backup, utilizar:

~~~yaml
spec:
  snapshotName: <SNAPSHOT>
~~~

Não utilizar spec.volume.

## DR combinado

Foi realizado teste com namespace, Deployment, Service, ConfigMap, Secret, PVC longhorn-prod e volume Longhorn.

Backup Velero:

~~~text
dr-test-combined
~~~

Backup Longhorn:

~~~text
backup-df6691ad8e4e450f
~~~

Volume original:

~~~text
pvc-4d7f4ce2-15ea-4cff-9b26-51797719fa2f
~~~

O namespace foi removido e a recuperação ocorreu em duas camadas:

~~~text
Velero
  ↓
Objetos Kubernetes

Longhorn
  ↓
Dados do volume
~~~

SHA-256 validado antes e depois:

~~~text
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
~~~

Os recursos de teste foram posteriormente removidos.

## Verificação

~~~bash
kubectl -n longhorn-system get nodes
kubectl -n longhorn-system get volumes.longhorn.io -o wide
kubectl -n longhorn-system get replicas.longhorn.io -o wide
kubectl -n longhorn-system get backuptarget default
kubectl -n longhorn-system get backupvolumes.longhorn.io
kubectl -n longhorn-system get backups.longhorn.io
kubectl -n longhorn-system get recurringjobs.longhorn.io
kubectl -n longhorn-system get cronjob backup-daily-r2
kubectl get storageclass
~~~

## Política operacional

- usar longhorn-prod nos PVCs que precisam da política de backup;
- manter backup diário às 02:00 UTC;
- retenção de 60 backups;
- utilizar Cloudflare R2;
- não versionar credenciais;
- executar testes periódicos de restore;
- monitorar falhas;
- revisar a decisão de 1 réplica antes de workloads críticos.

## Próximos passos

- monitoramento de falhas;
- teste periódico de restore;
- backup do sistema Longhorn;
- revisão da topologia física de storage;
- teste de perda de worker;
- teste de perda de host Proxmox;
- definição de RPO/RTO.
