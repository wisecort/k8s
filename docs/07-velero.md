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

```
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

```
https://<ACCOUNT_ID>.r2.cloudflarestorage.com
```

Não registrar credenciais no Git.

Verificar:

```bash
velero backup-location get
kubectl -n velero get backupstoragelocation default -o yaml
```

A validação do BackupStorageLocation foi concluída com sucesso.

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

A estratégia adotada para aplicações é:

### Velero

Backup dos objetos Kubernetes, incluindo conforme o escopo:

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
- CRs/CRDs quando incluídos;
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

## Teste de backup de objetos

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

O Velero processou 25 itens, porém o primeiro backup terminou como `Failed`.

### Falha inicial com Cloudflare R2

A causa foi identificada nos logs:

```
StatusCode: 501
NotImplemented:
Header 'x-amz-tagging' with value '' not implemented
```

O erro ocorreu durante a persistência dos artefatos do backup no R2.

A versão do plugin permaneceu:

```
velero/velero-plugin-for-aws:v1.14.2
```

Não foi aplicado release candidate em produção.

## Segundo backup — validado

Um novo backup foi executado posteriormente, mantendo o plugin `v1.14.2`:

```bash
velero backup create velero-test-02 \
  --include-namespaces velero-test \
  --wait
```

Resultado:

```
NAME              STATUS      ERRORS   WARNINGS
velero-test-02    Completed   0        0
```

Detalhes validados:

- 25 itens processados;
- 25 itens armazenados;
- 0 erros;
- 0 warnings;
- backup persistido no R2;
- retenção inicial do objeto: 720h.

O sucesso do segundo teste demonstra que o fluxo de backup de objetos Kubernetes para o R2 está operacional no ambiente atual. A causa da diferença entre o primeiro e o segundo teste não foi determinada e deve continuar sendo acompanhada antes de considerar a integração definitivamente encerrada.

## Restore de objetos Kubernetes

Foi realizado um teste de perda do namespace.

Namespace removido:

```bash
kubectl delete namespace velero-test
```

Restore executado:

```bash
velero restore create velero-restore-01 \
  --from-backup velero-test-02 \
  --wait
```

Resultado:

```
NAME                BACKUP           STATUS      ERRORS   WARNINGS
velero-restore-01   velero-test-02   Completed   0        1
```

Foram restaurados 12 itens.

Após o restore, foi validado:

```bash
kubectl -n velero-test get all
kubectl -n velero-test get configmap,secret
```

Resultado:

- Deployment `nginx`: 2/2 disponíveis;
- dois Pods: `Running`;
- ReplicaSet: 2 réplicas prontas;
- Service `nginx`: restaurado;
- ConfigMap `app-config`: restaurado;
- Secret `app-secret`: restaurado.

### Warning observado

O único warning foi referente ao ConfigMap:

```
kube-root-ca.crt
```

O recurso já existia no namespace restaurado e era diferente da cópia do backup. Trata-se de um recurso gerenciado pelo Kubernetes e o warning não impediu o restore da aplicação.

## Limitação atual

O teste acima valida somente **objetos Kubernetes**.

Ainda não foi validado o DR completo de uma aplicação com PVC.

A arquitetura definida mantém responsabilidades separadas:

```
Velero → objetos Kubernetes
Longhorn → dados dos PVCs
```

Portanto, não se deve considerar que um restore Velero sozinho restaura automaticamente o conteúdo dos volumes Longhorn.

O próximo teste de DR deverá combinar:

1. aplicação Kubernetes;
2. PVC usando `longhorn-prod`;
3. dados gravados no PVC;
4. backup dos objetos pelo Velero;
5. backup do volume pelo Longhorn/R2;
6. perda da aplicação;
7. restore dos objetos via Velero;
8. recuperação do volume via Longhorn;
9. validação dos dados da aplicação.

## Política de backup do Velero

A política inicial definida para objetos Kubernetes será:

- frequência: diária;
- horário: **04:00**;
- destino: Cloudflare R2 `velero-backups-qg`;
- escopo: objetos Kubernetes das aplicações;
- retenção inicial: **30 dias**;
- volumes: não utilizar Velero como backup primário dos dados dos PVCs;
- dados dos PVCs: responsabilidade do Longhorn/R2.

A política será implementada por meio de um `Schedule` do Velero.

Antes de considerar a política pronta para produção, validar:

```bash
velero schedule get
velero schedule describe <nome-do-schedule>
velero backup get
```

Também deverá ser validado pelo menos um backup criado pelo Schedule.

## Operação

Antes de considerar o Velero como componente de produção:

- [x] instalação;
- [x] BackupStorageLocation;
- [x] acesso ao R2;
- [x] backup de objetos Kubernetes;
- [x] restore de objetos Kubernetes;
- [ ] acompanhar estabilidade da integração R2/plugin;
- [ ] política de retenção de 30 dias;
- [ ] Schedule diário às 04:00;
- [ ] validar backup automático do Schedule;
- [ ] monitorar falhas;
- [ ] validar DR completo com PVC;
- [ ] documentar procedimento final de Disaster Recovery.
