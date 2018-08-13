# Info

Cluster mode **today** only works with flannel cni VXLAN mode.

## Setup VXLAN

- Create a VXLAN profile with flooding-type none

```bash
cd ../k8s-controller/
create /net tunnels vxlan fl-vxlan port 8472 flooding-type none
```

- Create a VXLAN tunnel
  - Set the local-address to an IP address from the network that will support the VXLAN overlay.
  - Set the key to 1 to grant the BIG-IP device access to all Cluster resources.

```bash
create /net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan local-address 172.16.30.3
```

- Identify the flannel subnet you want to assign to the BIG-IP system. Make sure it doesn’t overlap with a subnet that’s already in use by existing Nodes in the Kubernetes Cluster. You will assign this subnet to a “dummy” Node for the BIG-IP device later.
- Create a self IP using an address from the subnet you want to assign to the BIG-IP device.
  - The self IP range must fall within the cluster subnet mask. The flannel network’s default subnet mask is /16.
  - If you use the BIG-IP configuration utility to create a self IP, you may need to provide the full netmask instead of the CIDR notation.

```bash
create /net self 10.244.255.1/12 allow-service none vlan flannel_vxlan
```

- Create a floating IP address in the flannel subnet you assigned to the BIG-IP device.

```bash
create /net self 10.244.255.2/12 allow-service none traffic-group traffic-group-1 vlan flannel_vxlan
```

## All Commands

```bash
cd ../k8s-controller/
create /net tunnels vxlan fl-vxlan port 8472 flooding-type none
create /net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan local-address 172.16.30.3
create /net self 10.244.255.1/12 allow-service none vlan flannel_vxlan
create /net self 10.244.255.2/12 allow-service none traffic-group traffic-group-1 vlan flannel_vxlan
```

## Create Dummy Node (required)

```bash
kubectl apply -f f5-k8s-node/
```

## Configure Static Endpoints (Required inside ESXi)

- Actual mac addresses would need to be found on the K8S Nodes

  - On k8s nodes
    - ip -4 addr show
      - will show flannel interface
      - can see mac address here
    - ip -4 -d link show flannel.1
      - will also show mac-address
    - bridge fdb show dev flannel.1
      - shows forwarding-database for VXLAN
        - also has mac-address information

- These change after every kubeadm reset

```bash
modify net fdb tunnel flannel_vxlan records add { 46:03:26:d0:df:b8 { endpoint 172.16.30.11 } }
modify net fdb tunnel flannel_vxlan records add { d2:09:ce:e3:e4:75 { endpoint 172.16.30.21 } }
modify net fdb tunnel flannel_vxlan records add { ee:ef:e3:c5:c1:d3 { endpoint 172.16.30.22 } }
modify net fdb tunnel flannel_vxlan records add { be:4d:65:46:78:9e { endpoint 172.16.30.23 } }
```

- To delete the endpoints (if testing)

```bash
modify net fdb tunnel flannel_vxlan records del { 46:03:26:d0:df:b8 }
modify net fdb tunnel flannel_vxlan records del { d2:09:ce:e3:e4:75 }
modify net fdb tunnel flannel_vxlan records del { ee:ef:e3:c5:c1:d3 }
modify net fdb tunnel flannel_vxlan records del { be:4d:65:46:78:9e }
```

- verify VXLAN

```bash
show net tunnels tunnel flannel_vxlan all-properties
list net tunnels tunnel flannel_vxlan
show net fdb tunnel flannel_vxlan
```
