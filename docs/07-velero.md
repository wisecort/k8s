# Velero

## Versão

- Velero CLI/Server: 1.18.2
- Plugin AWS/S3: 1.14.2
- Kubernetes/RKE2: v1.36.4+rke2r1

## Objetivo

O Velero será utilizado para proteger o **estado Kubernetes das aplicações**, enquanto o Longhorn permanece responsável pelo backup recorrente dos dados dos PVCs.

A separação adotada é:

```
Objetos Kubernetes
  ↓
Velero
  ↓
Cloudflare R2

Dados dos PVCs
  ↓
Longhorn
  ↓
Cloudflare R2
```

O objetivo é evitar duplicação desnecessária dos dados dos volumes.

## Cloudflare R2

Foi criado um bucket separado para o Velero:

```
velero-backups-qg
```

O bucket é privado e possui credencial própria.

**Nunca armazenar Access Key, Secret Key ou tokens no Git.**

O Longhorn continua utilizando o bucket separado:

```
longhorn-backups-qg
```

## Instalação

O CLI foi instalado no macOS:

```bash
velero version --client-only
```

Resultado:

```
Client:
    Version: v1.18.2
```

O Velero foi instalado no cluster com o provider AWS/S3 e o plugin:

```
velero/velero-plugin-for-aws:v1.14.2
```

A instalação utilizou o endpoint S3-compatible do Cloudflare R2 e o bucket `velero-backups-qg`.

O `node-agent` também foi instalado para deixar o cluster preparado para File System Backup/Kopia, porém o File System Backup não foi habilitado como política padrão para os volumes.

## BackupStorageLocation

O BackupStorageLocation padrão é:

```text
Namespace: velero
Name: default
Provider: aws
Bucket: velero-backups-qg
Region: auto
Access Mode: ReadWrite
Default: true
Phase: Available
```

Endpoint:

```text
https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

Não registrar o Account ID/credenciais em documentação quando isso permitir identificar ou acessar recursos protegidos.

Verificar:

```bash
velero backup-location get
kubectl -n velero get backupstoragelocation default -o yaml
```

A validação do BackupStorageLocation foi concluída com sucesso:

```
PHASE: Available
ACCESS MODE: ReadWrite
DEFAULT: true
```

## Componentes

Verificar:

```bash
kubectl -n velero get pods -o wide
velero version
```

O cluster possui:

- Deployment `velero`;
- DaemonSet `node-agent`;
- um `node-agent` por nó.

Todos os componentes foram validados como `Running`.

## Estratégia de backup

A estratégia planejada para aplicações é:

### Velero

Backup dos objetos Kubernetes, incluindo conforme o escopo do backup:

- Deployments;
- StatefulSets;
- DaemonSets;
- Services;
- Ingresses;
- ConfigMaps;
- Secrets;
- ServiceAccounts;
- RBAC;
- Jobs;
- CronJobs;
- CRs/CRDs quando incluídos no escopo;
- PVC/PV metadata;
- demais recursos Kubernetes selecionados.

### Longhorn

Backup do conteúdo dos PVCs:

- snapshots;
- dados dos volumes;
- backups externos;
- restore de volumes.

Os PVCs de produção devem utilizar:

```yaml
storageClassName: longhorn-prod
```

## Teste inicial

Foi criado o namespace:

```
velero-test
```

Com:

- Deployment `nginx`;
- Service `nginx`;
- ConfigMap `app-config`;
- Secret `app-secret`.

O primeiro backup foi criado com:

```bash
velero backup create velero-test-01 \
  --include-namespaces velero-test \
  --wait
```

O Velero processou todos os recursos:

```
Total items to be backed up: 25
Items backed up: 25
```

Entretanto, o backup terminou como `Failed`.

## Falha encontrada com Cloudflare R2

A causa foi identificada nos logs do Deployment do Velero.

Erro:

```
StatusCode: 501
NotImplemented:
Header 'x-amz-tagging' with value '' not implemented
```

O erro ocorreu ao tentar gravar:

```
backups/velero-test-01/velero-test-01-logs.gz
```

e posteriormente:

```
backups/velero-test-01/velero-backup.json
```

Portanto:

- RKE2/Kubernetes: OK;
- Velero: OK;
- BackupStorageLocation: OK;
- autenticação no R2: OK;
- processamento dos objetos: OK;
- persistência final do backup no R2: falhou devido ao header `x-amz-tagging`.

O erro não envolve o Longhorn.

## Situação do plugin AWS

A versão atualmente instalada é:

```
velero/velero-plugin-for-aws:v1.14.2
```

O problema de `x-amz-tagging` foi identificado durante a integração com Cloudflare R2.

A estratégia atual é **não aplicar uma versão release candidate em produção sem validação**.

Foi criada uma tarefa de monitoramento externo para acompanhar a publicação de uma versão estável do `velero-plugin-for-aws` contendo a correção relacionada a:

```
Only set PutObject Tagging when tags are configured
```

Quando uma versão estável corrigida e compatível com Velero 1.18.2 estiver disponível, o plugin deverá ser atualizado e o teste de backup deverá ser repetido.

Verificar a imagem atualmente instalada:

```bash
kubectl -n velero get deployment velero \
  -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="velero-velero-plugin-for-aws")].image}{"\\n"}'
```

## Teste pendente

Depois que existir uma versão estável corrigida do plugin:

1. atualizar somente o initContainer do plugin AWS;
2. aguardar o rollout do Deployment;
3. criar novo backup do namespace `velero-test`;
4. validar `Completed`;
5. validar os objetos armazenados no R2;
6. remover o namespace de teste;
7. executar restore;
8. validar Deployment, Service, ConfigMap e Secret;
9. documentar o resultado.

Comandos previstos:

```bash
velero backup create velero-test-02 \
  --include-namespaces velero-test \
  --wait

velero backup get
velero backup describe velero-test-02 --details
```

## Restore

O restore do Velero ainda não foi considerado validado.

A validação final deverá demonstrar:

```
Backup
  ↓
Cloudflare R2
  ↓
delete namespace
  ↓
Velero Restore
  ↓
objetos Kubernetes restaurados
  ↓
aplicação funcionando
```

O restore de dados dos PVCs continua sendo responsabilidade do fluxo Longhorn/R2 já validado separadamente.

## Operação

Antes de considerar o Velero pronto para produção:

- resolver a incompatibilidade do plugin com R2;
- validar backup completo;
- validar restore;
- definir retenção;
- definir frequência;
- monitorar falhas;
- documentar procedimento de Disaster Recovery.

