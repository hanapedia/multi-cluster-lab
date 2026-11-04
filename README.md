# Multi-Cluster lab
A local lab environment for experimenting with the networking for multi-cluster configurations.

This project contains containerlab configurations for experimenting with different multi-cluster configurations locally.

## Invariable Configurations
### Failure domains
- 2 regions (west & east)
- 3 dc per region
- 1 cluster per dc

### Routing
- 1 global router container handling the routing between regions, representing the internet routing.
- each dc gets 1 router container handling routing within dc and between routers in other dc, representing the switches in each dc.
- routers in different dc are connected via VXLAN network
- routing information is exchanged via BGP

### Kubernetes clusters (kind clusters)

1 edge cluster per region
- responsible for handling the ingress from the outside network
- ingress is handled by L7 load balancer in the cluster

1 core clusters per region (for now)
- responsible for running the workload

Each cluster has 2 nodes (for now, may extend later)
Nodes have IPs from the same subnet but not reachable across clusters

### DNS
- Global DNS server at global router
- external-dns in each cluster writes to this DNS server to add records for their LoadBalancer IPs

## Variables
- Pod & Service CIDR overwrap
    - all clusters use the same CIDR (allow duplicate IPs across clusters)
    - unique CIDR per cluster (no duplicate IPs across clusters)
- egress path
    - egress is NAT/proxied at the edge clusters
    - egress is NAT/proxied at the originating core clusters
