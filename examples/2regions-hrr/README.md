# Example: 2 Regions with Hierarchical Route Reflectors
![Example Topology](../../assets/2regions-hrr.png)
![Example RR Hierarchy](../../assets/2regions-hrr-hierarchy.png)

## Description
* This topology has two regions, `mirkwood` and `lothlorian`, and a `core` region.
* Within a region, all cRPDs are fully meshed via iBGP to provide maximum visibility within the region.
* All cRPDs in a region other than `core` have BGP peerings with up to 2 `core` cRPDs as dictated by the `max_peers` setting. The `core` cRPDs serve as route reflectors for the non-core regions.
* **Redundancy groups and anycast addressing:**
  * Each node is assigned to redundancy group `a` or `b`. (In a production environment, two separate Kubernetes clusters may be desirable.)
  * For each region with neighbors outside the cluster, separate DaemonSets are created for `a` and `b`, each with a unique IP address for that [meshrr_region:redundancy_group] combination. This IP address is used for iBGP peering with neighbors outside the cluster.
  * Kubernetes nodes run MetalLB.
  * MetalLB eBGP peers to each connected router on a loopback with the same IP address (10.0.0.0).
  * meshrr pods are assigned to a Kubernetes service using the MetalLB load balancer. This service is configured with `externalTrafficPolicy: local` and MetalLB announces the /32 of the service only from nodes with a corresponding meshrr pod, which will act as a route reflector for routers outside the cluster.
  * This is the preferred method of distributing routes to the route reflector as it dynamically advertises based upon where route reflectors exist. It is possible, however, to use loopbacks with NodeIPs assigned to pods, but this lacks a native means of failover.
  * In the above topology:
    | Node           | MESHRR_REGION(s)     | Redundancy Group | MetalLB IP(s)          |
    | -------------- | -------------------- | ---------------- | ---------------------- |
    | lothlorien-vm1 | core, lothlorien     | a                | 172.19.1.1             |
    | lothlorien-vm2 | core, lothlorien     | b                | 172.19.1.2             |
    | lothlorien-vm3 | mirkwood, lothlorien | a                | 172.19.1.1, 172.19.2.1 |
    | lothlorien-vm4 | mirkwood, lothlorien | b                | 172.19.1.2, 172.19.2.2 |
    | mirkwood-vm1   | core, mirkwood       | a                | 172.19.2.1             |
    | mrikwood-vm2   | mirkwood             | b                | 172.19.2.2             |



### Usage
1.  Review the instructions in [the project's main README](../../README.md#Usage). Create a license secret:
    ```
    kubectl create secret generic crpd-license --from-file=crpd-license=<filepath>
    ```

2.  Configure the routers connected to the nodes with:
    *  The anycast peering address for MetalLB to peer to on the loopback
    *  Hardcoded router ID (to ensure that the anycast peering address does not become the router ID)
    *  BGP peering
    *  Redistribution of RR routes from BGP to IGP
    ```junos
    set interfaces lo0 unit 0 family inet address 10.0.0.0/32
    set protocols bgp group RR-LOADBALANCER type external
    set protocols bgp group RR-LOADBALANCER multihop ttl 2
    set protocols bgp group RR-LOADBALANCER local-address 10.0.0.0
    set protocols bgp group RR-LOADBALANCER peer-as 65001
    set protocols bgp group RR-LOADBALANCER allow 172.19.0.0/24
    set protocols bgp group RR-LOADBALANCER import NO-ADVERTISE
    set protocols isis export FILTER-RRLBPEER
    set protocols isis export REDISTRIBUTE-RRS
    set policy-options policy-statement FILTER-RRLBPEER from protocol direct route-filter 10/32 exact
    set policy-options policy-statement FILTER-RRLBPEER then reject
    set policy-options community no-advertise members no-advertise
    set policy-options policy-statement NO-ADVERTISE then community add no-advertise
    set policy-options policy-statement REDISTRIBUTE-RRS from protocol bgp route-filter 172.19.0.0/16 prefix-length-range /32-/32
    set policy-options policy-statement REDISTRIBUTE-RRS then accept
    ```

3.  Modify configuration templates as necessary. [`meshrr/juniper.conf.j2`](../../meshrr/defaults/juniper-ipv4rr.conf.j2) will be loaded to all instances by default, but customizations on a per-deployment/per-daemonset basis should be performed in most cases files (see [`mirkwood-config.j2`](templates/mirkwood-config.j2) and [`lothlorien-config.j2`](templates/lothlorien-config.j2)).
Apply these configuration templates as ConfigSets for any cases that require customization as so:

    ```bash
    kubectl create configmap mirkwood-config \
      --from-file=config=examples/2regions-hrr/templates/mirkwood-config.j2 \
      -o yaml --dry-run=client |
      kubectl apply -f -
    ```

    ```bash
    kubectl create configmap lothlorien-config \
      --from-file=config=examples/2regions-hrr/templates/lothlorien-config.j2 \
      -o yaml --dry-run=client |
      kubectl apply -f -
    ```

    These ConfigMaps are mounted as volumes in the corresponding Deployments/DaemonSets.

4.  Label the nodes. You may need to adjust the node names.
    ```bash
    kubectl label nodes lothlorien-vm1 meshrr_region_core="true" meshrr_region_lothlorien="true" redundancy_group=a
    kubectl label nodes lothlorien-vm2 meshrr_region_core="true" meshrr_region_lothlorien="true" redundancy_group=b
    kubectl label nodes lothlorien-vm3 meshrr_region_lothlorien="true" meshrr_region_mirkwood="true" redundancy_group=a
    kubectl label nodes lothlorien-vm4 meshrr_region_lothlorien="true" meshrr_region_mirkwood="true" redundancy_group=b
    kubectl label nodes mirkwood-vm1 meshrr_region_core="true" meshrr_region_mirkwood="true" redundancy_group=a
    kubectl label nodes mirkwood-vm2 meshrr_region_mirkwood="true" redundancy_group=b
    ```

5.  Apply the Kubernetes manifests:
    ```bash
    k apply -f examples/2regions-hrr/bgppeer-global.yml -f examples/2regions-hrr/meshrr-core.yaml -f examples/2regions-hrr/meshrr-lothlorien.yaml -f examples/2regions-hrr/meshrr-mirkwood.yaml
    ```

    See [Manifests and Objects Used](#Manifests-and-Objects-Used) for detail on what this does.

### Manifests and Objects Used

* [`bgppeer-global.yml`]()
  * `BGPPeer/asn65000-global-lo1`: Defines a single BGP peer that will be used from every node in the Kubernetes cluster out every interface. In this example, we use 10.0.0.0/32 as an anycast IP on every router that connects to the Kubernetes cluster. This must not be advertised in any routing protocol.
* [`meshrr-core.yaml`]()
  * `ConfigMap/meshrr-core-conf`: Contains the YAML to configure meshrr for the Core Deployment.
  * `Service/meshrr-core`: Headless service used for inter-pod BGP service discovery.
  * `Deployment/meshrr-core`: Defines the Deployment for the meshrr Core region.
* [`meshrr-lothlorien.yaml`]() and [`meshrr-mirkwood.yaml`]()
  * `ConfigMap/meshrr-<REGION>-conf`: Contains the YAML to configure meshrr for both the <REGION> DaemonSets.
  * `IPAddressPool/meshrr-<REGION>`: The address pool used for both the A side and B side load balancers in <REGION>.
  * `BGPAdvertisement/meshrr-<REGION>`: Configures MetalLB to advertise the load balancer addresses when a pod is reachable via a LoadBalancer service on the node.
  * `Service/meshrr-<REGION>-<RG>`: Configures a MetalLB LoadBalancer to connect external route reflector clients to a containerized route reflector in redundancy group <RG>. Note that `externalTrafficPolicy: Local` is of importance here to ensure that traffic to the service does not traverse the Kubernetes cluster to reach a Pod but only is advertised via BGP if a pod in the service exists on the node.
  * `DaemonSet/meshrr-<REGION>-<RG>`: The DaemonSet for the redundancy group <RG> nodes in  <REGION>. Note the use of DaemonSets vs Deployments: Deployments ensure that a certain number of pods exist within the cluster. DaemonSets ensure that a pod exists on each node within the cluster that matches the parameters defined in `nodeAffinity`.
