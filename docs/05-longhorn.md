# Longhorn

## Versão

1.12.1

## Nós de storage

- rke2-worker01
- rke2-worker02
- rke2-worker03

Os control planes estão com scheduling do Longhorn desabilitado.

## Storage

Data path:

`/var/lib/longhorn`

Cada worker possui disco dedicado para Longhorn.

## Réplicas

Configuração padrão:

`defaultReplicaCount: 3`

StorageClass padrão:

`longhorn`

O valor de réplica foi configurado de forma declarativa pelo Helm.

## StorageClass de produção

A StorageClass padrão `longhorn` foi mantida sem alterações para preservar o comportamento padrão do cluster.

Para workloads de produção foi criada a StorageClass:

`longhorn-prod`

Configuração:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-prod
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fromBackup: ""
  fsType: "ext4"
  dataLocality: "disabled"
  unmapMarkSnapChainRemoved: "ignored"
  disableRevisionCounter: "true"
  dataEngine: "v1"
  backupTargetName: "default"
  recurringJobSelector: '[{"name":"default","isGroup":true}]'
```

Novos PVCs de produção devem utilizar:

```yaml
spec:
  storageClassName: longhorn-prod
```

A StorageClass aplica o `recurringJobSelector` na criação de novos volumes.

Validação realizada com o volume:

`pvc-d281dc1f-6b33-4889-a640-4b92fc87290b`

O volume recebeu automaticamente o label:

```
recurring-job-group.longhorn.io/default=enabled
```

e foi validado como:

```
State: attached
Robustness: healthy
Number Of Replicas: 3
```

## Backup externo

O Longhorn utiliza **Cloudflare R2** como backup target S3-compatible.

Configuração lógica:

- Bucket: `longhorn-backups-qg`
- Região: `auto`
- Endpoint S3: configurado no Secret do Longhorn
- Backup target: `default`
- Caminho: `s3://longhorn-backups-qg@auto/backupstore`
- Secret: `cloudflare-r2-longhorn` no namespace `longhorn-system`

**Nunca armazenar Access Key, Secret Key ou tokens no Git.**

O Secret utiliza as credenciais S3 e o endpoint R2. Para Cloudflare R2, o endpoint deve ser informado através de `AWS_ENDPOINTS`, evitando que o Longhorn tente resolver o endpoint AWS padrão.

### Verificar o BackupTarget

```bash
kubectl -n longhorn-system get backuptarget default
kubectl -n longhorn-system get backuptarget default -o yaml
```

O campo `status.available` deve estar `true`.

## Backup manual

O fluxo validado foi:

```
PVC
  ↓
Longhorn Volume
  ↓
Snapshot
  ↓
Backup Full
  ↓
Cloudflare R2
```

O snapshot pode ser criado pela UI do Longhorn.

Listar snapshots:

```bash
kubectl -n longhorn-system get snapshots.longhorn.io
```

Depois, criar o backup a partir de um snapshot válido. O recurso `Backup` usa `spec.snapshotName`; não existe `spec.volume`.

Exemplo:

```yaml
apiVersion: longhorn.io/v1beta2
kind: Backup
metadata:
  name: exemplo-backup
  namespace: longhorn-system
spec:
  snapshotName: snap-EXEMPLO
  backupMode: full
```

Aplicar:

```bash
kubectl apply -f backup.yaml
```

Acompanhar:

```bash
kubectl -n longhorn-system get backups.longhorn.io -w
```

Validar:

```bash
kubectl -n longhorn-system get backups.longhorn.io -o wide
kubectl -n longhorn-system describe backup <BACKUP_NAME>
```

Um backup válido deve chegar a:

```
STATE: Completed
Progress: 100
Backup Target Name: default
```

O método de compressão utilizado nos testes foi LZ4.

## Teste de backup e restore realizado

Foi criado um PVC de 1Gi com um Pod de teste. Foram gravados:

- `teste.txt`
- `data-criacao.txt`
- `hostname.txt`
- `teste.bin` de 10 MiB

O conteúdo de `teste.txt` foi:

```
LONGHORN-R2-TEST
```

Foi criado um snapshot pelo Longhorn e executado um **Full Backup** para o R2.

Resultado do backup:

```
State: Completed
Progress: 100
Backup Target: default
Compression: lz4
```

O backup foi posteriormente restaurado criando uma StorageClass temporária com `parameters.fromBackup` apontando para a URL do backup.

Exemplo:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-r2-restore
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: "s3://<bucket>@auto/backupstore?backup=<backup>&volume=<volume>"
  fsType: "ext4"
```

Depois foi criado um novo PVC usando essa StorageClass.

O PVC restaurado ficou `Bound`, o volume foi montado em um Pod e os arquivos originais foram recuperados.

Validações realizadas:

```bash
kubectl exec r2-restore-pod -- sh -c 'ls -lh /data'
kubectl exec r2-restore-pod -- cat /data/teste.txt
kubectl exec r2-restore-pod -- sha256sum /data/teste.bin
```

O arquivo `teste.txt` retornou:

```
LONGHORN-R2-TEST
```

O arquivo `teste.bin` foi recuperado com 10 MiB.

Esse teste comprovou o fluxo:

```
Longhorn → Snapshot → Cloudflare R2 → Restore → PVC → Pod → dados
```

O parâmetro `fromBackup` é o mecanismo utilizado pelo Longhorn para restaurar um volume a partir de uma URL de backup.

## Backup recorrente

Foi criada uma política de backup recorrente para produção:

```yaml
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
```

Política:

- frequência: diária
- horário: 02:00
- retenção: 60 backups
- concorrência: 1
- tarefa: backup
- grupo: `default`
- destino: BackupTarget `default` → Cloudflare R2

O CronJob Kubernetes correspondente foi validado e uma execução manual foi realizada com sucesso.

Execução validada:

```
Job: backup-daily-r2-manual-1789947433
Status: Complete
Completions: 1/1
Duration: 29s
```

O backup produzido foi:

```
Backup: backup-40c3f6c4e3b9452d
State: Completed
Size: 67108864
Volume: pvc-d281dc1f-6b33-4889-a640-4b92fc87290b
```

O volume registrou:

```
LAST-BACKUP: backup-40c3f6c4e3b9452d
LAST-BACKUP-AT: 2026-09-20T23:37:21Z
STATE: attached
ROBUSTNESS: healthy
```

O RecurringJob registrou:

```
executionCount: 1
```

Isso comprovou o fluxo:

```
longhorn-prod
  ↓
recurring-job-group/default
  ↓
backup-daily-r2
  ↓
CronJob
  ↓
Longhorn Backup
  ↓
Cloudflare R2
```

Para workloads de produção, utilizar `longhorn-prod` como StorageClass para que novos volumes recebam automaticamente a política de backup recorrente.

## Política operacional

Para produção:

- usar `longhorn-prod` nos PVCs de aplicação;
- manter 3 réplicas;
- manter o backup recorrente diário;
- manter retenção de 60 backups;
- utilizar Cloudflare R2 como destino externo;
- monitorar falhas de backup;
- executar testes periódicos de restore;
- manter o backup do sistema Longhorn no plano de Disaster Recovery.

A retenção de 60 representa 60 execuções de backup mantidas; a duração efetiva da janela de histórico depende da frequência real de execução.

## Verificação

```bash
kubectl -n longhorn-system get nodes
kubectl -n longhorn-system get volumes -o wide
kubectl -n longhorn-system get replicas -o wide
kubectl -n longhorn-system get backuptarget default
kubectl -n longhorn-system get backupvolumes.longhorn.io
kubectl -n longhorn-system get backups.longhorn.io
kubectl -n longhorn-system get recurringjobs.longhorn.io
kubectl -n longhorn-system get cronjob backup-daily-r2
```

## Limpeza após testes

Os recursos Kubernetes usados somente para o teste podem ser removidos sem remover o backup já armazenado no R2:

```bash
kubectl delete pod r2-restore-pod r2-test-pod
kubectl delete pvc r2-restore-pvc r2-test-pvc
kubectl delete storageclass longhorn-r2-restore
```

**Atenção:** não remover o Backup/BackupVolume do Longhorn ou objetos do bucket R2 se o objetivo for preservar o ponto de recuperação do teste.

O PVC/Pod usado para validar a StorageClass `longhorn-prod` também pode ser removido após a validação:

```bash
kubectl delete pod backup-policy-test
kubectl delete pvc backup-policy-test
```

O backup já armazenado no R2 não depende desses recursos Kubernetes permanecerem no cluster.

## Próximo passo

- backup periódico do sistema Longhorn;
- monitoramento e alertas de falha;
- teste periódico de restore;
- integração com o plano de Disaster Recovery.
