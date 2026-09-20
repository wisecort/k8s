# DNS

## Zona interna

`k8s.internal`

## Authoritative DNS

PowerDNS:

10.255.10.143

Registros principais:

| Nome | IP |
|---|---|
| k8s-api.k8s.internal | 10.255.10.140 |
| lb01.k8s.internal | 10.255.10.141 |
| lb02.k8s.internal | 10.255.10.142 |
| cp01.k8s.internal | 10.255.10.130 |
| cp02.k8s.internal | 10.255.10.131 |
| cp03.k8s.internal | 10.255.10.132 |
| worker01.k8s.internal | 10.255.10.133 |
| worker02.k8s.internal | 10.255.10.134 |
| worker03.k8s.internal | 10.255.10.135 |

## Recursive DNS

10.255.10.10

PowerDNS é authoritative e não deve ser usado como resolver recursivo geral pelos nós Kubernetes.

O resolver recursivo precisa encaminhar a zona `k8s.internal` para o PowerDNS authoritative quando aplicável.
