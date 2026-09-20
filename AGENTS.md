# AGENTS.md

## Objetivo

Este repositório documenta a plataforma Kubernetes baseada em RKE2 executada sobre Proxmox.

A documentação deve refletir o estado real do ambiente. Não registrar credenciais, tokens, chaves privadas ou outros segredos.

## Regras

1. Preferir configuração declarativa e reproduzível.
2. Registrar mudanças relevantes no documento do componente.
3. Não alterar componentes de produção sem verificar dependências e impacto.
4. Não armazenar secrets reais no Git.
5. Preferir Helm values e manifests versionados a patches manuais.
6. Toda mudança estrutural deve atualizar a documentação correspondente.
7. Em caso de dúvida, verificar o estado atual do cluster antes de assumir uma configuração.
8. Não confundir esta plataforma RKE2 com os projetos OKD antigos.

## Plataforma atual

- Kubernetes distribution: RKE2
- Kubernetes: v1.36.4+rke2r1
- CNI: Canal
- Control planes: 3
- Workers: 3
- Storage: Longhorn 1.12.1
- Longhorn replicas: 3
- API VIP: 10.255.10.140
- API DNS: k8s-api.k8s.internal

## Estado

Concluído:
- RKE2 HA
- 3 control planes
- 3 workers
- HAProxy
- Keepalived
- VIP do Kubernetes API
- DNS interno
- kubeconfig via VIP
- Longhorn com 3 réplicas
- teste funcional de PVC

Próximos componentes:
- backup externo do Longhorn
- exposição pública via Traefik/HAProxy
- cert-manager
- segurança multi-tenant
- observabilidade
- GitOps
- disaster recovery
