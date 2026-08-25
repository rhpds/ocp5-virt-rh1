# Module 06 — Templates and InstanceType Management

### Brief Overview

This module explores how OpenShift Virtualization standardizes VM provisioning through two complementary mechanisms: VM templates (which bundle OS configuration, boot source, and resource settings into a reusable catalog item) and instance types (which define compute profiles — CPU and memory — independently of the OS). OpenShift 5 formally adopts instance types as the preferred sizing model, deprecating the prior approach of specifying resources directly in the VM spec. Participants explore built-in cluster instance types, create a custom instance type, deploy a VM using that instance type, and build a custom VM template that can be shared across the cluster catalog.

### Audience and Time

- **Personas:** Platform engineers, infrastructure architects, lab administrators responsible for maintaining a VM catalog
- **Prerequisites for this module:** Module 01 completed; CLI access available
- **Estimated duration:** 20 minutes

### Learning Objectives

- Explore the built-in cluster instance types (`u1.*`, `cx1.*`, `m1.*` series) and their CPU/memory profiles
- Create a custom VirtualMachineClusterInstancetype to define a site-specific compute profile
- Deploy a virtual machine referencing the custom instance type and verify the applied resource profile
- Build a custom VirtualMachineClusterPreference and combine it with the instance type to create a parameterized VM catalog entry
- Manage the cluster VM template catalog by creating a namespaced template and promoting it for cluster-wide visibility

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Exploring Built-in Instance Types and Preferences | 5 min |
| 2 | Creating a Custom Instance Type | 5 min |
| 3 | Deploying a VM from an Instance Type | 5 min |
| 4 | Creating a Custom VM Template | 5 min |

### Detailed Steps

**Section 1 — Exploring Built-in Instance Types and Preferences**

1. Navigate to **Virtualization → Catalog** in the web console. Click the **InstanceTypes** tab. Browse the pre-installed cluster instance types — note the `u1` (general purpose), `cx1` (compute-intensive), and `m1` (memory-optimized) series introduced in OpenShift 5.
2. Click `u1.small` to view its spec: 1 CPU, 2 GiB memory. Note that instance types only define compute resources — storage and network are defined separately.
3. From the CLI, list all cluster-scoped instance types:
   ```
   oc get virtualmachineclusterinstancetype
   ```
4. Inspect a specific instance type in detail:
   ```
   oc get virtualmachineclusterinstancetype u1.medium -o yaml
   ```
   Note the `spec.cpu.guest` and `spec.memory.guest` fields — these are the only fields in a VirtualMachineClusterInstancetype spec.
5. List cluster-scoped preferences (OS-level defaults bundled with OS images):
   ```
   oc get virtualmachineclusterpreference
   ```
   Describe `fedora` preference to see default disk bus, network interface model, and clock settings that OpenShift 5 applies when creating Fedora VMs.

**Section 2 — Creating a Custom Instance Type**

6. Create a custom cluster instance type representing a small application server profile. Apply the following YAML:
   ```yaml
   apiVersion: instancetype.kubevirt.io/v1beta1
   kind: VirtualMachineClusterInstancetype
   metadata:
     name: app-small
   spec:
     cpu:
       guest: 2
     memory:
       guest: 4Gi
   ```
   ```
   oc apply -f app-small-instancetype.yaml
   ```
7. Confirm the new instance type is available:
   ```
   oc get virtualmachineclusterinstancetype app-small
   ```
8. Create a namespace-scoped instance type (visible only within your namespace) for comparison:
   ```yaml
   apiVersion: instancetype.kubevirt.io/v1beta1
   kind: VirtualMachineInstancetype
   metadata:
     name: dev-micro
     namespace: <your-namespace>
   spec:
     cpu:
       guest: 1
     memory:
       guest: 512Mi
   ```
   ```
   oc apply -f dev-micro-instancetype.yaml
   ```
   Observe the distinction: `VirtualMachineClusterInstancetype` is cluster-wide; `VirtualMachineInstancetype` is namespace-scoped.

**Section 3 — Deploying a VM from an Instance Type**

9. Create a VM that references the `app-small` cluster instance type and the `fedora` cluster preference. Apply the following YAML:
   ```yaml
   apiVersion: kubevirt.io/v1
   kind: VirtualMachine
   metadata:
     name: app-server-01
     namespace: <your-namespace>
   spec:
     running: true
     instancetype:
       kind: VirtualMachineClusterInstancetype
       name: app-small
     preference:
       kind: VirtualMachineClusterPreference
       name: fedora
     dataVolumeTemplates:
       - metadata:
           name: app-server-01-root
         spec:
           storage:
             resources:
               requests:
                 storage: 10Gi
             storageClassName: ocs-storagecluster-ceph-rbd
           source:
             pvc:
               namespace: openshift-virtualization-os-images
               name: fedora-latest
     template:
       spec:
         domain:
           devices: {}
         volumes:
           - name: app-server-01-root
             dataVolume:
               name: app-server-01-root
   ```
   ```
   oc apply -f app-server-01.yaml
   ```
10. Verify the VM starts and confirm the applied instance type is reflected in the VM spec:
    ```
    oc get vm app-server-01 -n <your-namespace> -o jsonpath='{.spec.instancetype}'
    ```
11. Note in OCP5 that the instance type spec is snapshotted into a `ControllerRevision` at VM creation time, ensuring the VM's compute profile is immutable even if the instance type CR is later modified:
    ```
    oc get controllerrevision -n <your-namespace> | grep app-server
    ```

**Section 4 — Creating a Custom VM Template**

12. Navigate to **Virtualization → Catalog → Templates** in the web console. Click **Create Template**.
13. In the OCP5 template editor, set:
    - **Template name:** `rhel9-app-server`
    - **Base template:** `rhel9` (select from the list)
    - **Instance type:** `app-small`
    - **Storage:** 20 GiB, `ocs-storagecluster-ceph-rbd`
    - Add a template **label:** `app-tier=backend`
14. Click **Create Template**. The template is created in your namespace.
15. View the template YAML from the CLI:
    ```
    oc get template rhel9-app-server -n <your-namespace> -o yaml
    ```
16. (Optional) To make the template available cluster-wide, it would be created in the `openshift` namespace (requires cluster-admin rights). In this lab, observe the template in your namespace only.
17. Create a VM from the template in the web console: navigate to **Virtualization → Catalog**, filter by your namespace, locate `rhel9-app-server`, and click **Quick create VirtualMachine**.

### Key Takeaways

- OpenShift 5 standardizes on instance types as the recommended compute sizing model, separating the "what compute" decision (instance type) from the "what OS" decision (preference and boot source).
- Instance types are immutable at VM creation time through ControllerRevision snapshots — this prevents accidental drift if the cluster instance type definition changes later.
- Cluster-scoped instance types and preferences form a shared catalog that administrators define once and all teams consume, promoting standardization across a platform.
- Custom VM templates combine an instance type, OS preference, storage profile, and optional parameters into a reusable catalog item — the equivalent of a cloud provider AMI or a hypervisor VM template.
- Namespace-scoped instance types and templates allow teams to manage their own catalog items without cluster-admin rights.
