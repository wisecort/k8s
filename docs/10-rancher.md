# Rancher

## Estado

- Rancher: 2.15.1
- Kubernetes/RKE2: v1.36.4+rke2r1
- Namespace: cattle-system
- Hostname: rancher.k8s.internal
- Réplicas atuais: 1
- IngressClass: traefik
- TLS: cert-manager com CA interna
- Fleet: operacional

## Arquitetura

O Rancher roda dentro do próprio cluster RKE2 e gerencia o cluster local.

~~~text
rancher.k8s.internal
        |
        v
      Traefik
        |
        v
     Rancher
        |
        v
   RKE2 local
~~~

O Rancher não foi publicado diretamente na Internet.

DNS interno:

~~~text
10.255.10.133
10.255.10.134
10.255.10.135
~~~

O VIP 10.255.10.140 continua dedicado ao Kubernetes API/supervisor através de HAProxy.

## TLS

PKI interna via cert-manager.

CA:

~~~text
Certificate: rancher-ca
Namespace: cert-manager
Secret: rancher-ca-key-pair
CN: Rancher Internal CA
RSA: 4096
~~~

Issuer:

~~~text
ClusterIssuer: rancher-ca
~~~

Certificado Rancher:

~~~text
Certificate: rancher-tls
Namespace: cattle-system
Secret: rancher-tls
CN: rancher.k8s.internal
DNS SAN: rancher.k8s.internal
RSA: 2048
Issuer: rancher-ca
~~~

Validade observada:

~~~text
NotBefore: 2026-09-21T02:03:51Z
NotAfter:  2026-12-20T02:03:51Z
Renewal:   2026-11-20T02:03:51Z
~~~

## Instalação

~~~bash
helm upgrade --install rancher rancher-stable/rancher   --namespace cattle-system   --create-namespace   --version 2.15.1   --set hostname=rancher.k8s.internal   --set ingress.tls.source=secret   --set ingress.tls.secretName=rancher-tls   --set replicas=1   --set-string bootstrapPassword="$RANCHER_BOOTSTRAP_PASSWORD"   --wait   --timeout 15m
~~~

A senha bootstrap não é armazenada no Git.

## Componentes validados

~~~text
rancher                     1/1
rancher-webhook             1/1
system-upgrade-controller   1/1

fleet-controller            1/1
gitjob                      1/1
helmops                     1/1
~~~

Não havia pods em CrashLoopBackOff, Error, Pending ou NotReady na validação final.

## Ingress

~~~text
NAME      CLASS     HOSTS                  PORTS
rancher   traefik   rancher.k8s.internal   80,443
~~~

O campo ADDRESS pode permanecer vazio neste desenho porque o Traefik utiliza hostPort nos nós.

## Teste de conectividade

~~~bash
dig +short rancher.k8s.internal
~~~

Retornou os três workers.

~~~bash
curl -vk https://rancher.k8s.internal
~~~

Resultado validado:

~~~text
HTTP/2 200
~~~

O endpoint respondeu com a API do Rancher.

O -k foi utilizado porque a CA interna ainda não estava instalada como CA confiável no macOS.

## Escala

O Rancher está com uma réplica de propósito.

Antes de aumentar para 3 réplicas, considerar a instabilidade de I/O observada no host Proxmox pve2.

Depois de estabilizar a camada física de storage, avaliar:

~~~bash
helm upgrade rancher rancher-stable/rancher   --namespace cattle-system   --reuse-values   --set replicas=3   --wait   --timeout 15m
~~~

## Operação

~~~bash
kubectl get deployment -n cattle-system
kubectl get pods -n cattle-system
kubectl get ingress -n cattle-system
kubectl get certificate -n cattle-system
kubectl get pods -n cattle-fleet-system
~~~

## Segurança

- não versionar bootstrapPassword;
- não versionar tokens;
- não versionar Secrets;
- manter Rancher interno;
- utilizar TLS;
- revisar RBAC;
- instalar a CA interna nas estações administrativas;
- trocar a senha inicial após o primeiro acesso.

## Próximos passos

- [ ] instalar/trustar a CA interna no macOS
- [ ] Rancher Monitoring
- [ ] configurar alertas
- [ ] Argo CD
- [ ] GitOps
- [ ] avaliar 3 réplicas após estabilização de storage/I/O
