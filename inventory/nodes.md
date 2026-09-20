# Inventário de nós

## Control Plane

| Node | VMID | IP | vCPU | RAM |
|---|---:|---|---:|---:|
| rke2-cp01 | 130 | 10.255.10.130 | 4 | 12 GB |
| rke2-cp02 | 131 | 10.255.10.131 | 4 | 12 GB |
| rke2-cp03 | 132 | 10.255.10.132 | 4 | 12 GB |

## Workers

| Node | VMID | IP | vCPU | RAM |
|---|---:|---|---:|---:|
| rke2-worker01 | 133 | 10.255.10.133 | 4 | 12 GB |
| rke2-worker02 | 134 | 10.255.10.134 | 4 | 12 GB |
| rke2-worker03 | 135 | 10.255.10.135 | 4 | 12 GB |

## Load Balancers

| Node | VMID | IP | Papel |
|---|---:|---|---|
| LB01 | 141 | 10.255.10.141 | MASTER |
| LB02 | 142 | 10.255.10.142 | BACKUP |

VIP: 10.255.10.140

## Rede

- VLAN: 97
- Gateway: 10.255.10.1
- Recursive DNS: 10.255.10.10
- PowerDNS authoritative: 10.255.10.143
