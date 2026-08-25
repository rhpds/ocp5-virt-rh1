# Module 05 — Backup and Recovery with OADP

### Brief Overview

This module introduces OpenShift API for Data Protection (OADP) as a comprehensive backup and recovery solution for virtual machines in OpenShift Virtualization. While Module 04 covered VM-level snapshots, OADP provides cluster-aware backup that captures the full VM definition, associated PVCs, and related Kubernetes objects in a single backup artifact stored in an S3-compatible object store. Participants configure a backup location, create a backup of a running VM, simulate a failure by deleting the VM, and perform a full restore. OpenShift 5 ships with an updated OADP integration that supports VM-consistent backup using the kubevirt OADP plugin, which is highlighted in this module.

### Audience and Time

- **Personas:** Platform engineers, virtualization administrators, infrastructure architects responsible for VM data protection
- **Prerequisites for this module:** Module 02 completed; a running or stopped VM (`fedora-lab`) in the participant namespace; OADP operator installed; an S3-compatible object storage bucket pre-configured in the lab environment
- **Estimated duration:** 20 minutes

### Learning Objectives

- Configure a BackupStorageLocation in OADP to target the lab's S3-compatible object store
- Create a backup of a VirtualMachine and its associated storage using the OADP kubevirt plugin
- Verify backup contents and confirm the backup artifact is stored in the object store
- Restore a deleted VirtualMachine from an OADP backup and verify it returns to a running state

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Exploring the OADP Operator and Configuration | 5 min |
| 2 | Creating a Backup | 7 min |
| 3 | Simulating Failure and Restoring | 8 min |

### Detailed Steps

**Section 1 — Exploring the OADP Operator and Configuration**

1. Navigate to **Operators → Installed Operators** and click **OADP Operator**. Confirm the operator is in the `openshift-adp` namespace and shows `Succeeded` status.
2. Click the **DataProtectionApplication** tab and inspect the pre-created `dpa-lab` resource. Note the configuration sections:
   - `backupLocations` — references the S3 bucket credentials secret
   - `snapshotLocations` — references the CSI snapshot provider (ODF in this lab)
   - `plugins` — confirms `kubevirt` plugin is enabled (an OCP5 addition for VM-aware backup)
3. From the CLI, verify the BackupStorageLocation is available:
   ```
   oc get backupstoragelocation -n openshift-adp
   ```
   Confirm the `dpa-lab-1` location shows `Phase: Available`.
4. Check that the OADP VolumeSnapshotLocation is configured:
   ```
   oc get volumesnapshotlocation -n openshift-adp
   ```

**Section 2 — Creating a Backup**

5. Ensure `fedora-lab` is running in your namespace. Write a marker file in the VM to verify it survives restore:
   In the VM console:
   ```
   echo "oadp-backup-marker" | sudo tee /tmp/backup-marker.txt
   ```
6. Create a Backup CR targeting your namespace. Apply the following YAML (substituting your namespace):
   ```yaml
   apiVersion: velero.io/v1
   kind: Backup
   metadata:
     name: fedora-lab-backup-01
     namespace: openshift-adp
   spec:
     includedNamespaces:
       - <your-namespace>
     labelSelector:
       matchLabels:
         vm.kubevirt.io/name: fedora-lab
     storageLocation: dpa-lab-1
     volumeSnapshotLocations:
       - dpa-lab-1
     hooks:
       resources:
         - name: kubevirt-vm-freeze
           includedNamespaces:
             - <your-namespace>
           labelSelector:
             matchLabels:
               vm.kubevirt.io/name: fedora-lab
           pre:
             - exec:
                 container: ""
                 command:
                   - /usr/bin/bash
                   - -c
                   - ""
   ```
   Apply with:
   ```
   oc apply -f backup.yaml -n openshift-adp
   ```
   Note: In OCP5 with the kubevirt OADP plugin, the plugin automatically quiesces the VM filesystem before snapshotting — no manual freeze hooks are required.
7. Monitor backup progress:
   ```
   oc get backup fedora-lab-backup-01 -n openshift-adp -w
   ```
   Wait for `Phase: Completed`. This typically takes 2–3 minutes.
8. Inspect the backup object for completeness:
   ```
   oc describe backup fedora-lab-backup-01 -n openshift-adp
   ```
   Confirm the items backed up include the VirtualMachine, VirtualMachineInstance (if running), DataVolume, and PVC objects.
9. (Optional) Use the `velero` CLI to list backup contents:
   ```
   velero backup describe fedora-lab-backup-01 --details -n openshift-adp
   ```

**Section 3 — Simulating Failure and Restoring**

10. Delete the VM and its storage to simulate accidental deletion:
    ```
    oc delete vm fedora-lab -n <your-namespace>
    oc delete dv fedora-lab -n <your-namespace>
    oc delete pvc fedora-lab -n <your-namespace>
    ```
11. Confirm the VM is gone:
    ```
    oc get vm -n <your-namespace>
    oc get pvc -n <your-namespace>
    ```
    Both commands should return no resources.
12. Create a Restore CR to recover from the backup:
    ```yaml
    apiVersion: velero.io/v1
    kind: Restore
    metadata:
      name: fedora-lab-restore-01
      namespace: openshift-adp
    spec:
      backupName: fedora-lab-backup-01
      includedNamespaces:
        - <your-namespace>
    ```
    Apply with:
    ```
    oc apply -f restore.yaml -n openshift-adp
    ```
13. Monitor restore progress:
    ```
    oc get restore fedora-lab-restore-01 -n openshift-adp -w
    ```
    Wait for `Phase: Completed`.
14. Verify the VM reappears in your namespace:
    ```
    oc get vm -n <your-namespace>
    oc get pvc -n <your-namespace>
    ```
15. Start the restored VM:
    ```
    virtctl start fedora-lab -n <your-namespace>
    ```
16. Open the VM console and confirm the marker file survived the restore:
    ```
    cat /tmp/backup-marker.txt
    ```
    The output `oadp-backup-marker` confirms the restore included the VM's filesystem state at backup time.

### Key Takeaways

- OADP provides a cluster-level backup solution that captures the full set of Kubernetes objects associated with a VM — not just disk data — enabling complete recreation of a VM in a new cluster or namespace.
- The kubevirt OADP plugin in OpenShift 5 enables VM-consistent backup by coordinating with the KubeVirt control plane to quiesce guests before snapshotting, eliminating the need for manual freeze scripts.
- OADP complements (rather than replaces) VM-level snapshots: snapshots are fast, local, and ideal for rollback; OADP backups are portable, stored externally, and suitable for disaster recovery.
- Backup and restore operations target Kubernetes objects by label selector, making it straightforward to back up a single VM, a namespace, or a fleet.
- Restore creates new object instances from backup metadata — the original objects do not need to exist, making OADP suitable for full disaster recovery scenarios.

### Infrastructure Notes

- The lab environment pre-configures an S3-compatible object store (e.g., ODF MCG/Noobaa) and a secret with bucket credentials in `openshift-adp`. Participants do not need to create the bucket or credentials.
- The `velero` CLI binary is available in the lab terminal. It is not required for basic backup/restore operations (which can be done entirely via CRs) but is useful for diagnostics.
