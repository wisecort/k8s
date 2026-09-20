# Rede e VIP

## Kubernetes API

- VIP: 10.255.10.140
- DNS: k8s-api.k8s.internal
- API: TCP/6443
- RKE2 supervisor: TCP/9345

## Load Balancers

HAProxy roda em LB01 e LB02.

Keepalived fornece o VIP 10.255.10.140 com:
- LB01 priority 110
- LB02 priority 100

O failover do VIP já foi testado com sucesso.

## HAProxy

Backends do API e supervisor:

- cp01 10.255.10.130
- cp02 10.255.10.131
- cp03 10.255.10.132

Stats: TCP 8404.

## Observação

O health tracking de HAProxy pelo Keepalived ainda é uma melhoria futura.
