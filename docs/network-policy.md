# Network Policy for traffic across clusters

## Premise
- Pod CIDR and Service CIDR overlaps between clusters
- Inter-Cluster communication require NAT and L4LB
- NetworkPolicy is enforced by Cilium
    - Cilium >= 1.17 (required for `cidrGroupSelector`; older versions are EoL anyway)
- L4LB runs in DSR mode so the client's source (egress) IP is preserved up to the backend pod
    - without it, `externalTrafficPolicy: Cluster` SNATs to the node IP and the remote identity is lost

## Core idea
- enforce network policy by assigning egress and ingress IPs per CiliumIdentity
    - these IPs must not overlap between clusters and must be routable across clusters
    - these IPs can be more than one each
- CiliumCIDRGroup (CCG) is created in each remote cluster for those IPs and referenced in CiliumNetworkPolicy (CNP)
    - these CCG should have labels matching the security labels of the remote CiliumIdentity
    - CNP reference CCG by those labels via `cidrGroupSelector` in `fromCIDRSet` / `toCIDRSet`

## Missing Pieces
- A coordinating controllers in each cluster that create CCG in their own cluster with the ingress and egress IPs of CiliumIdentities in the remote cluster.
    - should not be every CiliumIdentity, only the ones opted-in. probably by a label on pods
    - full mesh between clusters to communicate about new identities that should be mirrored.
    - peering between clusters must be authenticated
        - the cluster label (`io.cilium.k8s.policy.cluster`) on mirrored CCGs should be derived from the authenticated peer, not taken from the peer's payload
- A controller that assigns ingress and egress IPs to each CiliumIdentity in their own cluster.
    - egress IP can be allocated via mooring (my WIP egress gateway solution. see github.com/hanapedia/mooring) 
        - or coil egress, but may not be feasible since IPs are coupled to namespaces
    - ingress IP can be allocated via Cilium's LBIPAM
    - requirement: IPs released by a deleted identity must not be reallocated until all remote clusters have removed the corresponding CCG (or a cooldown has passed)
        - otherwise the new identity's traffic is attributed to the old identity's labels in remote clusters
- must use cluster indicating security label `k8s:io.cilium.k8s.policy.cluster=<cluster-name>` by specifying `--cluster-name`
