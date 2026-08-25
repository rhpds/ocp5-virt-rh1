# Module 04 — VM Storage Management

### Brief Overview

This module explores how OpenShift Virtualization integrates with Kubernetes-native persistent storage for virtual machines. Participants work with DataVolumes and PersistentVolumeClaims as VM disk backing, create point-in-time snapshots, restore a VM from a snapshot, and clone a VM by cloning its underlying disk. OpenShift Data Foundation (ODF) is the storage backend in this lab. OpenShift 5 improves the snapshot and clone workflows through tighter console integration and introduces volume population from external sources as a first-class feature. Understanding the storage model is critical for VM portability, backup strategies, and efficient multi-VM deployments.

### Audience and Time

- **Personas:** Platform engineers, virtualization administrators, infrastructure architects
- **Prerequisites for this module:** Module 02 completed; a running VM (`fedora-lab`) in the participant namespace; ODF storage classes available
- **Estimated duration:** 20 minutes

### Learning Objectives

- Configure additional storage for a VM by adding a new DataVolume disk via the console and CLI
- Create a VirtualMachine snapshot and verify its contents
- Restore a VM to a previous state using a snapshot
- Clone a virtual machine by cloning its root disk DataVolume to provision a second VM

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Exploring VM Storage Resources | 4 min |
| 2 | Adding a DataVolume Disk to a Running VM | 4 min |
| 3 | Creating and Inspecting a Snapshot | 6 min |
| 4 | Restoring from a Snapshot | 6 min |

### Detailed Steps

**Section 1 — Exploring VM Storage Resources**

1. Navigate to **Virtualization → VirtualMachines** and click `fedora-lab`. Click the **Disks** tab. Note the root disk — in OCP5 it is backed by a DataVolume named after the VM, with the storage class shown inline.
2. From the CLI, inspect the DataVolume backing the root disk:
   ```
   oc get dv -n <your-namespace>
   oc get pvc -n <your-namespace>
   ```
   Note that the DataVolume controller reconciles a PVC on your behalf; the PVC name matches the DataVolume name.
3. Describe the DataVolume to see its source, capacity, and phase:
   ```
   oc describe dv fedora-lab -n <your-namespace>
   ```
   Confirm the phase is `Succeeded` and the storage class is `ocs-storagecluster-ceph-rbd` (or the equivalent ODF block storage class in the lab cluster).
4. Check the VolumeSnapshotClass available for this storage class:
   ```
   oc get volumesnapshotclass
   ```
   Confirm an ODF-backed VolumeSnapshotClass exists — this is required for VM snapshots.

**Section 2 — Adding a DataVolume Disk to a Running VM**

5. In the `fedora-lab` VM detail view, click the **Disks** tab, then **Add disk**.
6. Configure the new disk:
   - **Source:** Blank
   - **Name:** `data-disk`
   - **Size:** 5 GiB
   - **Storage class:** `ocs-storagecluster-ceph-rbd`
   - **Access mode:** ReadWriteOnce
7. Click **Save**. The disk is hot-plugged into the running VM (OCP5 supports hot-plug without a VM restart for block volumes).
8. Open the VM console and verify the new disk is visible:
   ```
   lsblk
   ```
   You should see a new block device (e.g., `/dev/vdb`). Format and mount it:
   ```
   sudo mkfs.ext4 /dev/vdb
   sudo mkdir /mnt/data
   sudo mount /dev/vdb /mnt/data
   echo "snapshot test" | sudo tee /mnt/data/testfile.txt
   ```

**Section 3 — Creating and Inspecting a Snapshot**

9. Stop the VM before taking a snapshot to ensure filesystem consistency. In the web console: **Actions → Stop**, or:
   ```
   virtctl stop fedora-lab -n <your-namespace>
   ```
10. Navigate to the **Snapshots** tab on the VM detail page. Click **Take snapshot**.
11. Set **Snapshot name:** `fedora-lab-snap-01`. Leave the description optional. Click **Save**.
12. Monitor the snapshot status — it transitions from `InProgress` to `Succeeded`. In OCP5 the snapshot progress percentage is shown in the console.
13. From the CLI, inspect the VirtualMachineSnapshot object:
    ```
    oc get vmsnapshot -n <your-namespace>
    oc describe vmsnapshot fedora-lab-snap-01 -n <your-namespace>
    ```
    Note the `readyToUse: true` condition and the list of included volumes.
14. Inspect the underlying VolumeSnapshot resources created per disk:
    ```
    oc get volumesnapshot -n <your-namespace>
    ```

**Section 4 — Restoring from a Snapshot**

15. Simulate accidental data loss. Start the VM, open the console, and delete the test file:
    ```
    virtctl start fedora-lab -n <your-namespace>
    ```
    In the VM console:
    ```
    sudo rm /mnt/data/testfile.txt
    ls /mnt/data/
    ```
    Confirm the file is gone.
16. Stop the VM again before restoring:
    ```
    virtctl stop fedora-lab -n <your-namespace>
    ```
17. In the web console **Snapshots** tab, click the three-dot menu next to `fedora-lab-snap-01` and select **Restore VirtualMachine snapshot**.
18. Confirm the restore action. The VM status transitions to `Restoring`, then to `Stopped`.
19. Start the VM and re-mount the data disk in the console:
    ```
    sudo mount /dev/vdb /mnt/data
    cat /mnt/data/testfile.txt
    ```
    Confirm `snapshot test` is present, confirming a successful restore.
20. From the CLI, view the VirtualMachineRestore object created by the restore operation:
    ```
    oc get vmrestore -n <your-namespace>
    ```

### Key Takeaways

- DataVolumes are the OpenShift Virtualization abstraction that manages PVC lifecycle on behalf of a VM — they handle import, cloning, and provisioning in a single object.
- OpenShift 5 supports hot-plugging block storage volumes into running VMs, reducing the need for maintenance windows when adding capacity to a VM.
- VM snapshots in OpenShift Virtualization use the Kubernetes VolumeSnapshot API, making snapshot management consistent with any CSI-compliant storage provider.
- Snapshot and restore is a critical first step in implementing a data protection strategy; Module 05 extends this with full backup and recovery using OADP.
- ODF's Ceph RBD storage class provides efficient copy-on-write snapshot and clone semantics, making these operations fast regardless of disk size.

### Infrastructure Notes

- ODF must be installed and a VolumeSnapshotClass backed by the Ceph CSI driver must exist for VM snapshots to function. The lab cluster pre-configures both.
- Hot-plug requires the `hotplugVolumes` feature gate enabled in the HyperConverged CR — this is enabled by default in OCP5.
