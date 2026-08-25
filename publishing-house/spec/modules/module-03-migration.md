# Module 03 — Migrating Existing VMs with MTV

### Brief Overview

This module covers the Migration Toolkit for Virtualization (MTV), which enables participants to migrate virtual machines from external hypervisors — such as VMware vSphere — into OpenShift Virtualization. Participants configure an MTV provider, define network and storage mappings, create a migration plan, execute the migration, and validate the resulting VM on OpenShift 5. The module targets the most common real-world scenario for virtualization administrators evaluating or transitioning to OpenShift: bringing existing VM workloads into the platform without rebuilding them. OpenShift 5 includes updated MTV integration with improved migration status reporting and support for warm migration via CBT.

### Audience and Time

- **Personas:** Virtualization administrators, infrastructure architects, platform engineers evaluating VM consolidation
- **Prerequisites for this module:** Module 01 completed; MTV operator confirmed installed; a source provider (simulated vSphere or pre-configured RHV/OVA source) available in the lab environment
- **Estimated duration:** 30 minutes

### Learning Objectives

- Configure a source provider in MTV to connect to an external hypervisor
- Create network and storage mappings that translate external hypervisor resources to OpenShift equivalents
- Build and execute a migration plan to migrate a VM into OpenShift Virtualization
- Verify the migrated VM boots correctly and is accessible in OpenShift 5
- Analyze migration status and troubleshoot common migration failures using MTV's status reporting

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Exploring the MTV Operator and Console | 5 min |
| 2 | Adding a Source Provider | 7 min |
| 3 | Creating Network and Storage Mappings | 5 min |
| 4 | Creating and Running a Migration Plan | 8 min |
| 5 | Validating the Migrated VM | 5 min |

### Detailed Steps

**Section 1 — Exploring the MTV Operator and Console**

1. In the OpenShift 5 web console, navigate to **Migration → Providers for Virtualization**. Confirm the MTV operator is installed and the console extension is active (the Migration menu item appears in the left sidebar).
2. Observe the default **Host** provider — this represents the local OpenShift cluster as the migration target. Note that in OCP5 the host provider is automatically registered at operator installation time.
3. Review the **Plans**, **Providers**, and **NetworkMaps/StorageMaps** menu items to understand the MTV workflow structure.
4. From the CLI, confirm the MTV CRDs are available:
   ```
   oc get crd providers.forklift.konveyor.io
   oc get crd plans.forklift.konveyor.io
   ```

**Section 2 — Adding a Source Provider**

5. Click **Providers for Virtualization → Create Provider**. Select the provider type matching the lab source (e.g., **VMware** for a simulated vSphere endpoint, or **Open Virtual Appliance (OVA)** if using a pre-staged OVA file).
6. For OVA provider (lab default):
   - Set **Name:** `lab-ova-source`
   - Set **URL:** point to the OVA staging PVC path shown in the lab panel (e.g., `ova://pvc/lab-ova`)
   - Leave credentials blank (OVA provider does not require authentication)
   - Click **Create**
7. Wait for the provider status to show **Ready**. This may take 30–60 seconds as MTV inventories the OVA.
8. Click on the provider name to inspect the inventory — confirm the source VM (`rhel-webserver`) appears in the virtual machine list.

**Section 3 — Creating Network and Storage Mappings**

9. Navigate to **Migration → NetworkMaps for Virtualization → Create NetworkMap**.
10. Set:
   - **Name:** `lab-network-map`
   - **Source provider:** `lab-ova-source`
   - **Target provider:** `host` (the local OCP5 cluster)
   - Map the source network (e.g., `VM Network`) to the target network (`Pod Networking`)
   - Click **Create**
11. Navigate to **Migration → StorageMaps for Virtualization → Create StorageMap**.
12. Set:
   - **Name:** `lab-storage-map`
   - **Source provider:** `lab-ova-source`
   - **Target provider:** `host`
   - Map the source datastore to the target storage class (e.g., `ocs-storagecluster-ceph-rbd`)
   - Click **Create**
13. Confirm both maps show **Ready** status before proceeding.

**Section 4 — Creating and Running a Migration Plan**

14. Navigate to **Migration → Plans for Virtualization → Create Plan**.
15. Complete the wizard:
   - **Plan name:** `lab-migration-plan`
   - **Source provider:** `lab-ova-source`
   - **Target namespace:** your assigned namespace
   - **Network map:** `lab-network-map`
   - **Storage map:** `lab-storage-map`
   - **VMs:** select `rhel-webserver`
   - Leave migration type as **Cold** (the default; warm migration requires CBT-enabled vSphere sources)
   - Click **Create Plan**
16. On the plan detail page, click **Start Migration**. Confirm the action in the dialog.
17. Monitor the migration pipeline stages in the plan's **Virtual Machines** tab:
    - `AllocateDisks` → `CopyDisks` → `CreateVM` → `Completed`
18. Click on the VM row in the plan to view per-stage logs. Note the disk copy throughput shown in OCP5's improved migration status view.
19. From the CLI, watch the migration job progress:
    ```
    oc get plan lab-migration-plan -n <your-namespace> -o jsonpath='{.status.conditions}' | python3 -m json.tool
    ```

**Section 5 — Validating the Migrated VM**

20. When the plan status shows **Completed**, navigate to **Virtualization → VirtualMachines** in your namespace. The `rhel-webserver` VM should appear.
21. Click the VM name and start it if it is not already running: **Actions → Start**.
22. Open the **Console** tab and log in to confirm the VM has booted successfully. Verify the hostname and network interface configuration match the source VM metadata.
23. From the CLI, confirm the VMI is running and check its IP address:
    ```
    oc get vmi rhel-webserver -n <your-namespace> -o jsonpath='{.status.interfaces[*].ipAddress}'
    ```
24. Inspect the DataVolume created by MTV to understand how disk data was imported:
    ```
    oc get dv -n <your-namespace>
    ```
25. Note the annotations on the VM object that MTV adds to record migration provenance (source provider, plan name, migration timestamp).

### Key Takeaways

- MTV abstracts the complexity of VM migration through a declarative provider/mapping/plan model that translates hypervisor-specific resources to OpenShift-native equivalents.
- Network and storage mappings are reusable across multiple migration plans, which accelerates large-scale migration projects.
- Cold migration copies disks while the source VM is powered off; warm migration (CBT) allows the source to remain running during most of the copy, minimizing downtime.
- OpenShift 5's updated MTV console integration provides per-stage pipeline visibility and disk copy throughput metrics, making it easier to track and troubleshoot migrations.
- After a successful migration, the VM is a first-class OpenShift Virtualization VM — it benefits from all OCP5 lifecycle operations (live migration, snapshots, backup) with no special configuration.

### Infrastructure Notes

- The lab uses a pre-staged OVA source to simulate a VMware environment without requiring a live vSphere connection. The OVA is stored as a PVC in the `lab-infra` namespace.
- The OVA provider type is appropriate for isolated lab environments; production migrations typically use the vSphere or RHV provider types with live hypervisor credentials.
