# Disaster Recovery — Restore Combinado

## Objetivo

Recuperar uma aplicação após perda dos objetos Kubernetes e do volume persistente.

~~~text
Velero   → objetos Kubernetes
Longhorn → dados do PVC
~~~

## Cenário validado

Namespace: dr-test

Deployment: dr-test  
Image: nginx:1.29-alpine  
Replicas: 1

PVC: dr-test-pvc  
StorageClass: longhorn-prod  
AccessMode: ReadWriteOnce  
Size: 1Gi

Dados:

~~~text
/data/dr-test.txt
/data/data-criacao.txt
/data/hostname.txt
/data/teste.bin
/data/teste.bin.sha256
~~~

SHA-256 antes do desastre:

~~~text
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
~~~

## Backup Velero

~~~text
Backup: dr-test-combined
Velero-Native Snapshot PVs: false
CSI Snapshots: none
Pod Volume Backups: none
~~~

## Backup Longhorn

~~~text
Backup: backup-df6691ad8e4e450f
Volume: pvc-4d7f4ce2-15ea-4cff-9b26-51797719fa2f
State: Completed
Progress: 100
Compression: lz4
Backup Target: default
~~~

## Simulação

~~~bash
kubectl delete namespace dr-test
~~~

O namespace, PVC e volume original foram removidos.

## Restore Velero

~~~bash
velero restore create dr-test-combined-restore   --from-backup dr-test-combined   --restore-volumes=false   --wait
~~~

Os objetos Kubernetes foram restaurados.

O PVC recriado inicialmente apontava para um novo volume vazio. Esse volume não deve ser considerado o dado original.

## Restore Longhorn

Foi utilizada uma StorageClass temporária com fromBackup:

~~~yaml
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
~~~

O volume restaurado foi validado com 3 réplicas e estado saudável.

## Validação

~~~bash
kubectl -n dr-test exec deploy/dr-test -- sha256sum /data/teste.bin
~~~

Resultado:

~~~text
69da2dd94af7335e8635d4301a78e43ef9e86abc1f2f54667792f8c40fb1026b
~~~

O SHA-256 foi idêntico ao registrado antes da destruição.

## Runbook

1. confirmar backup Velero;
2. confirmar backup Longhorn;
3. restaurar objetos com Velero sem volumes;
4. identificar PVCs que precisam de dados;
5. evitar ou remover PVC vazio criado automaticamente;
6. restaurar volume pelo Longhorn;
7. associar PVC restaurado ao workload;
8. iniciar aplicação;
9. validar saúde;
10. validar dados críticos;
11. registrar evidências.

## Estado após o teste

O namespace dr-test, PVC e volume de teste foram posteriormente removidos. Os backups externos permanecem no R2.

## Limitações

- perda de worker;
- perda de host Proxmox;
- perda de control plane;
- perda simultânea de múltiplos workers;
- recuperação completa do cluster;
- StatefulSets complexos;
- cenários RWX;
- automação completa;
- RPO/RTO formal.
