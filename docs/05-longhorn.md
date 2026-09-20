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

## Teste realizado

Foi criado um PVC de 1Gi e um Pod de teste.

Resultado:
- volume attached
- robustness healthy
- 3 réplicas running
- uma réplica em cada worker
- arquivo escrito e lido com sucesso

O PVC e o Pod de teste foram removidos após a validação.

## Verificação

```bash
kubectl -n longhorn-system get nodes
kubectl -n longhorn-system get volumes -o wide
kubectl -n longhorn-system get replicas -o wide
```

## Próximo passo

Configurar backup externo S3-compatible e executar teste de restore.
