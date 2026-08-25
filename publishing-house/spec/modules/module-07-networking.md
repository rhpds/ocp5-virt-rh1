# Module 07 — VM Networking

### Brief Overview

This module covers how virtual machines connect to networks in OpenShift Virtualization: the default pod network, secondary interfaces using Network Attachment Definitions (NADs), and the User-Defined Networks (UDN) feature introduced in OpenShift 5. By default, VMs join the pod network and receive a cluster-internal IP address via the Multus primary CNI. For workloads requiring direct Layer 2 connectivity, VLAN tagging, or isolation between tenant networks, secondary interfaces backed by NADs provide the necessary flexibility. OpenShift 5's User-Defined Networks further simplify multi-tenant network segmentation by letting namespace owners define isolated Layer 3 networks without cluster-admin involvement.

### Audience and Time

- **Personas:** Platform engineers, infrastructure architects, network administrators evaluating OpenShift networking for VM workloads
- **Prerequisites for this module:** Module 02 completed; a running VM (`fedora-lab`) in the participant namespace; the SR-IOV or Linux bridge CNI plugin available on cluster nodes (lab cluster pre-configured)
- **Estimated duration:** 25 minutes

### Learning Objectives

- Explore the default pod network configuration for a virtual machine and identify its network interface model
- Create a Network Attachment Definition (NAD) to define a secondary Linux bridge network for VMs
- Configure a VM with a secondary network interface backed by the NAD and verify Layer 2 connectivity
- Demonstrate User-Defined Networks (OCP5) by creating an isolated tenant network scoped to a namespace
- Analyze network topology differences between pod networking, secondary interfaces, and user-defined networks

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Exploring Default Pod Network on a VM | 5 min |
| 2 | Creating a Network Attachment Definition | 5 min |
| 3 | Attaching a Secondary Interface to a VM | 7 min |
| 4 | User-Defined Networks in OpenShift 5 | 8 min |

### Detailed Steps

**Section 1 — Exploring Default Pod Network on a VM**

1. Ensure `fedora-lab` is running in your namespace. Navigate to its detail page and click the **Network Interfaces** tab. Note the default interface named `default` with the `masquerade` binding method — this is how OpenShift Virtualization connects a VM to the pod network using NAT.
2. Open the VM console and inspect the network interface:
   ```
   ip addr show
   ip route
   ```
   Note the VM has a single interface (e.g., `eth0`) with a `10.0.2.0/24` address — this is the internal masquerade address. The pod itself receives the cluster pod network IP.
3. From the CLI, inspect the virt-launcher pod's network configuration:
   ```
   oc get pod -n <your-namespace> -l vm.kubevirt.io/name=fedora-lab -o wide
   ```
   Note the pod IP. The VM's traffic egresses through the pod network via masquerade NAT.
4. Verify the VM can reach the cluster DNS and external internet through the pod network:
   In the VM console:
   ```
   curl -s https://www.redhat.com -o /dev/null -w "%{http_code}\n"
   ```

**Section 2 — Creating a Network Attachment Definition**

5. Create a NAD using the Linux bridge CNI plugin. This defines a secondary bridge network named `vm-bridge-net` in your namespace:
   ```yaml
   apiVersion: k8s.cni.cncf.io/v1
   kind: NetworkAttachmentDefinition
   metadata:
     name: vm-bridge-net
     namespace: <your-namespace>
   spec:
     config: |
       {
         "cniVersion": "0.3.1",
         "name": "vm-bridge-net",
         "type": "cnv-bridge",
         "bridge": "br-lab",
         "vlan": 0,
         "macspoofchk": true
       }
   ```
   Apply with:
   ```
   oc apply -f vm-bridge-nad.yaml
   ```
6. Verify the NAD is created:
   ```
   oc get network-attachment-definitions -n <your-namespace>
   ```
7. In the web console, navigate to **Virtualization → VirtualMachines → fedora-lab → Network Interfaces**. Confirm the NAD `vm-bridge-net` appears as an option when adding a new interface.

**Section 3 — Attaching a Secondary Interface to a VM**

8. Add a secondary interface to the running VM. In OCP5, secondary interfaces can be hot-plugged without stopping the VM. In the **Network Interfaces** tab, click **Add network interface**.
9. Configure the new interface:
   - **Name:** `secondary-nic`
   - **Network:** `vm-bridge-net` (select the NAD created in the previous section)
   - **Type:** Bridge
   - **MAC address:** leave blank to auto-assign
10. Click **Save**. The interface is added to the running VM. Note the `Pending changes` banner in OCP5 — hot-plugged network interfaces take effect immediately in KubeVirt 1.x (OCP5), unlike prior versions where a VM restart was required.
11. In the VM console, verify the new interface appeared:
    ```
    ip addr show
    ```
    You should see a new interface (e.g., `eth1`) without an IP address (the bridge does not provide DHCP by default). Assign a static IP for testing:
    ```
    sudo ip addr add 192.168.100.10/24 dev eth1
    sudo ip link set eth1 up
    ```
12. From the CLI, inspect the VMI spec to confirm the secondary interface is recorded:
    ```
    oc get vmi fedora-lab -n <your-namespace> -o jsonpath='{.status.interfaces}' | python3 -m json.tool
    ```

**Section 4 — User-Defined Networks in OpenShift 5**

13. OpenShift 5 introduces User-Defined Networks (UDN), which allow namespace owners to create isolated Layer 3 networks without cluster-admin access. Create a UserDefinedNetwork CR in your namespace:
    ```yaml
    apiVersion: k8s.ovn.org/v1
    kind: UserDefinedNetwork
    metadata:
      name: tenant-net
      namespace: <your-namespace>
    spec:
      topology: Layer3
      layer3:
        role: Primary
        subnets:
          - cidr: 10.200.0.0/16
            hostSubnet: 24
    ```
    Apply with:
    ```
    oc apply -f tenant-udn.yaml
    ```
14. Confirm the UDN is active:
    ```
    oc get userdefinednetwork tenant-net -n <your-namespace>
    ```
    Wait for the `NetworkReady` condition to become `True`.
15. Create a second test VM and attach it to the UDN by setting the primary network to `tenant-net` in the VM spec. In OCP5, VMs can select their primary network via the `spec.template.spec.networks` field referencing a namespace-scoped UDN.
16. Once both VMs are on the `tenant-net`, test Layer 3 connectivity between them using ping — traffic stays within the namespace-isolated network and does not traverse the shared pod network.
17. Observe that pods in other namespaces cannot reach the `10.200.0.0/16` subnet — UDN provides hard network isolation without VLANs or external network infrastructure.
18. From the CLI, review the NetworkAttachmentDefinition auto-generated by the UDN controller:
    ```
    oc get network-attachment-definitions -n <your-namespace>
    ```
    Note that OCP5's UDN controller automatically provisions the underlying NAD, eliminating manual CNI configuration for tenant networks.

### Key Takeaways

- OpenShift Virtualization supports three networking models for VMs: pod network (masquerade NAT), secondary interfaces via NADs (Layer 2 bridge or SR-IOV), and User-Defined Networks (OCP5 isolated Layer 3 namespaced networks).
- Masquerade binding is the simplest and most portable option — VMs reach the cluster network and external internet through NAT with no additional configuration.
- Network Attachment Definitions expose the full Multus CNI ecosystem to VMs, enabling VLAN-backed bridges, SR-IOV passthrough, and other advanced networking topologies.
- User-Defined Networks are a key OCP5 networking innovation: they provide per-namespace network isolation with a self-service API, reducing the need for cluster-admin involvement in tenant network setup.
- OCP5 supports hot-plugging secondary network interfaces into running VMs, enabling network reconfiguration without VM downtime.

### Infrastructure Notes

- The lab cluster has the Linux bridge CNI (`cnv-bridge`) deployed on all worker nodes. The bridge `br-lab` is pre-created on each node as part of lab automation.
- User-Defined Networks require the OVN-Kubernetes CNI with the UDN feature gate enabled — this is the default CNI for OpenShift 5 and the feature is enabled by default.
