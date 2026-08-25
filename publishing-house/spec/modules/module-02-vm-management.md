# Module 02 — Virtual Machine Management

### Brief Overview

This module covers the full VM lifecycle within OpenShift Virtualization: creating a VM from a boot source, starting and stopping it, accessing its console, performing live migration between nodes, and managing the VM with `virtctl`. Participants work with both the web console and the CLI to understand both interaction models. OpenShift 5 introduces improved VM creation workflows with a streamlined wizard and enhanced live migration controls, which are highlighted during the hands-on steps. By the end of this module, participants will have created, run, and interactively managed their first VM on OpenShift 5.

### Audience and Time

- **Personas:** Platform engineers, virtualization administrators
- **Prerequisites for this module:** Module 01 completed; CLI access verified; dedicated namespace available
- **Estimated duration:** 30 minutes

### Learning Objectives

- Create a virtual machine from a pre-staged boot source using the OpenShift 5 console wizard
- Create a virtual machine using OpenShift Lightspeed natural language prompts via the MCP server
- Manage VM lifecycle states (start, stop, restart, pause) via both the console and `virtctl`
- Access a running VM using the web console VNC terminal and `virtctl console`
- Demonstrate live migration of a VM between cluster nodes with zero downtime
- Troubleshoot a VM issue using OpenShift Lightspeed in the console
- Explore the VirtualMachine and VirtualMachineInstance resource model in OpenShift 5

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Creating a VM from the Console | 6 min |
| 2 | Creating a VM with OpenShift Lightspeed | 7 min |
| 3 | VM Lifecycle Operations | 5 min |
| 4 | Accessing the VM Console | 5 min |
| 5 | Live Migration | 7 min |
| 6 | Inspecting VM Resources with the CLI | 3 min |
| 7 | Troubleshooting with Lightspeed | 5 min |

### Detailed Steps

**Section 1 — Creating a VM from the Console**

1. In the OpenShift 5 web console, navigate to **Virtualization → VirtualMachines** and ensure your namespace is selected in the project dropdown.
2. Click **Create VirtualMachine**. In OCP5 the creation wizard opens directly to the **Quick create** view — note that you no longer need to navigate through a separate catalog page.
3. Select the **Fedora** boot source from the template list. Observe the pre-staged PVC boot source badge indicating no download is required.
4. Set the VM name to `fedora-lab` and accept the default instance type (e.g., `u1.small`). Note that OCP5 defaults to instance types for all new VMs rather than raw resource specifications — this enforces consistent sizing standards.
5. Click **Create VirtualMachine**. The console transitions to the VM detail page and the VM begins starting automatically.

**Section 2 — Creating a VM with OpenShift Lightspeed**

6. In the web console, open the **OpenShift Lightspeed** assistant panel (the chat icon in the top navigation bar).
7. Ask Lightspeed to create a VM: type a natural language prompt such as:
   > "Create a Fedora virtual machine named `fedora-lightspeed` in my namespace with 2 CPUs and 4GB of memory."
8. Lightspeed uses the OCP5 MCP server integration to translate the prompt into a `VirtualMachine` manifest. Review the generated YAML in the Lightspeed response panel.
9. Click **Apply** (or copy and apply with `oc apply -f -`) to create the VM from the Lightspeed-generated manifest.
10. Confirm the VM appears in **Virtualization → VirtualMachines** and starts successfully.
11. Compare the two VMs (`fedora-lab` and `fedora-lightspeed`) — note that both use instance types, demonstrating that Lightspeed follows OCP5 best practices in its generated output.

**Section 3 — VM Lifecycle Operations**

6. Observe the VM status indicator cycle from `Provisioning` → `Starting` → `Running`. Note the Events tab for provisioning activity.
7. Click **Actions → Stop** to stop the VM. Confirm the status transitions to `Stopped`.
8. Click **Actions → Start** to restart the VM. Wait for `Running` status.
9. From the embedded terminal, stop the VM using the CLI:
   ```
   virtctl stop fedora-lab -n <your-namespace>
   ```
10. Start it again using the CLI:
    ```
    virtctl start fedora-lab -n <your-namespace>
    ```
11. Pause the VM and observe the `Paused` status:
    ```
    virtctl pause vm fedora-lab -n <your-namespace>
    ```
    Unpause it:
    ```
    virtctl unpause vm fedora-lab -n <your-namespace>
    ```

**Section 3 — Accessing the VM Console**

12. In the VM detail view, click the **Console** tab. The VNC console loads in-browser. Log in with the default Fedora credentials (`fedora` / `fedora` or the seed credentials shown in the console).
13. Run a command inside the VM to confirm the guest is responsive:
    ```
    uname -r
    hostname
    ```
14. Return to the terminal tab. Open a serial console with `virtctl`:
    ```
    virtctl console fedora-lab -n <your-namespace>
    ```
    Press `Ctrl+]` to exit the serial console.
15. Open an SSH session using `virtctl ssh` (OCP5 feature — no external IP required):
    ```
    virtctl ssh fedora@fedora-lab -n <your-namespace>
    ```
    Confirm the connection and exit with `exit`.

**Section 4 — Live Migration**

16. In the web console VM detail view, click **Actions → Migrate**. Note the new OCP5 migration confirmation dialog which shows the current node and estimated target selection.
17. Observe the VM status change to `Migrating`. Click the **Events** tab to watch the live migration progress in real time.
18. Confirm migration completes and the VM returns to `Running` on the new node. The **Details** tab shows the updated node name.
19. Trigger a live migration from the CLI:
    ```
    virtctl migrate fedora-lab -n <your-namespace>
    ```
    Watch the migration:
    ```
    oc get vmim -n <your-namespace> -w
    ```

**Section 5 — Inspecting VM Resources with the CLI**

20. Inspect the VirtualMachine object:
    ```
    oc get vm fedora-lab -n <your-namespace> -o yaml
    ```
    Note the `spec.instancetype` reference — in OCP5, instance types are stored as a separate CR rather than inlined into the VM spec.
21. Inspect the running VirtualMachineInstance:
    ```
    oc get vmi fedora-lab -n <your-namespace> -o wide
    ```
    Note the `nodeName` field reflecting the node the VM is currently scheduled on.
22. Check the VM's underlying pod (the virt-launcher pod):
    ```
    oc get pod -n <your-namespace> -l vm.kubevirt.io/name=fedora-lab
    ```

**Section 7 — Troubleshooting with Lightspeed**

29. Intentionally misconfigure a VM by editing its spec to reference a non-existent boot source:
    ```
    oc patch vm fedora-lab -n <your-namespace> --type=merge \
      -p '{"spec":{"dataVolumeTemplates":[{"metadata":{"name":"fedora-lab"},"spec":{"source":{"http":{"url":"http://invalid-url/fedora.qcow2"}},"storage":{"resources":{"requests":{"storage":"30Gi"}}}}}]}}'
    ```
    Delete and recreate the VM to trigger the error state.
30. Open the Lightspeed assistant and describe the problem: "My VM `fedora-lab` is stuck in `Provisioning` — can you help me diagnose it?"
31. Lightspeed queries live cluster resources via the MCP server and returns a diagnosis pointing to the failed DataVolume import. Review the suggested fix.
32. Restore the VM to working order using Lightspeed's suggested `oc` commands or by reverting the patch.

### Key Takeaways

- OpenShift 5 defaults to instance types for VM sizing, promoting consistency and simplifying VM creation — a shift from the OCP4 approach of specifying CPU and memory directly in the VM spec.
- A VirtualMachine (VM) object represents desired state; a VirtualMachineInstance (VMI) represents the running guest — the distinction matters for lifecycle management.
- Live migration in OpenShift Virtualization moves running VMs between nodes with no downtime, using the same mechanism underpinning Kubernetes pod scheduling.
- `virtctl` provides VM-native CLI operations (console, SSH, migrate, start/stop) that are not available through standard `oc` commands alone.
- The OCP5 console's unified Virtualization section and improved wizard reduce the steps needed to get a VM running compared to prior versions.
- OpenShift Lightspeed with MCP server integration allows natural language VM creation and live cluster diagnostics — Lightspeed generates OCP5-compliant manifests and can query running resources to assist with troubleshooting.
