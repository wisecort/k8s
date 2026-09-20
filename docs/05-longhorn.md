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

StorageClass:

`longhorn`

O valor de réplica foi configurado de forma declarativa pelo Helm.

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

### Backup manual

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

O Longhorn suporta blocos de backup de 2 MiB e 16 MiB. Nesta plataforma foi utilizado o padrão de 2 MiB. O método de compressão utilizado no teste foi LZ4. citeturn0search2turn0search1

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

O parâmetro `fromBackup` é o mecanismo documentado pelo Longhorn para restaurar um volume a partir de uma URL de backup. citeturn0search3turn0search6

## Restore de emergência

Para restaurar um volume sem utilizar o volume original:

1. Identificar o backup no BackupTarget.
2. Obter a URL S3 do backup.
3. Criar uma StorageClass temporária com `fromBackup`.
4. Criar um PVC apontando para essa StorageClass.
5. Aguardar o PVC ficar `Bound`.
6. Montar o PVC em um Pod.
7. Validar os arquivos e, quando aplicável, comparar hashes.
8. Somente depois substituir/reconfigurar a aplicação para usar o volume restaurado.

O Longhorn também documenta recuperação de backups sem o sistema Longhorn instalado, inclusive para gerar imagens `raw` ou `qcow2`. citeturn0search5

## Backup recorrente

Para produção, o backup manual não deve ser o mecanismo principal.

O Longhorn recomenda utilizar **Recurring Backup Jobs** para volumes críticos e manter o backup em um objeto externo. A documentação recomenda pelo menos um backup recorrente para cada volume que precisa de proteção. citeturn0search4

A política de produção deverá definir:

- frequência do backup;
- retenção;
- janela de execução;
- quantidade de backups mantidos;
- política de snapshots;
- monitoramento de falhas;
- teste periódico de restore.

Antes de aplicar uma política global, validar a necessidade de cada workload.

## Verificação

```bash
kubectl -n longhorn-system get nodes
kubectl -n longhorn-system get volumes -o wide
kubectl -n longhorn-system get replicas -o wide
kubectl -n longhorn-system get backuptarget default
kubectl -n longhorn-system get backupvolumes.longhorn.io
kubectl -n longhorn-system get backups.longhorn.io
```

## Limpeza após testes

Os recursos Kubernetes usados somente para o teste podem ser removidos sem remover o backup já armazenado no R2:

```bash
kubectl delete pod r2-restore-pod r2-test-pod
kubectl delete pvc r2-restore-pvc r2-test-pvc
kubectl delete storageclass longhorn-r2-restore
```

**Atenção:** não remover o Backup/BackupVolume do Longhorn ou objetos do bucket R2 se o objetivo for preservar o ponto de recuperação do teste.

## Próximo passo

Configurar **Recurring Backup Jobs**, retenção e política operacional de backup para os volumes de produção.

Também manter o backup do próprio sistema Longhorn como parte do plano de Disaster Recovery. A documentação do Longhorn trata backup de volumes e backup do sistema como mecanismos complementares. citeturn0search0turn0search4
