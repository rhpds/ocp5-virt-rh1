# Module 01 — Introduction and Environment Overview

### Brief Overview

This module orients participants to the lab environment and establishes baseline familiarity with the OpenShift 5 web console and CLI tools before any hands-on VM work begins. Participants verify that the OpenShift Virtualization operator is running, explore the Virtualization section of the console, and confirm access to `oc` and `virtctl`. Because OpenShift 5 consolidates several console navigation paths compared to OCP4, this orientation is essential for participants new to the updated UI. The module is deliberately short — its sole purpose is confidence-building before the hands-on modules that follow.

### Audience and Time

- **Personas:** Platform engineers, infrastructure architects, virtualization administrators
- **Prerequisites for this module:** None — this is the entry point
- **Estimated duration:** 15 minutes

### Learning Objectives

- Explore the OpenShift 5 web console and locate Virtualization-specific views
- Verify that the OpenShift Virtualization operator and required CRDs are installed and healthy
- Configure CLI access by logging into the cluster with `oc` and installing `virtctl`
- Identify the dedicated namespace assigned for the duration of the lab

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Accessing the Lab Environment | 3 min |
| 2 | Touring the OpenShift 5 Console | 5 min |
| 3 | CLI Tool Setup and Verification | 4 min |
| 4 | Namespace Orientation | 3 min |

### Detailed Steps

1. Open the lab portal URL provided in the showroom environment panel and note your assigned username, password, and OpenShift console URL.
2. Log in to the OpenShift 5 web console using the credentials provided. Observe the updated OCP5 navigation sidebar — note that **Virtualization** now appears as a top-level menu item rather than nested under Workloads.
3. Click **Virtualization** in the left navigation. Review the Overview dashboard: confirm that the HyperConverged custom resource shows `Available` status and that the virt-operator, virt-controller, and virt-handler pods are listed as healthy.
4. Click **VirtualMachines** under Virtualization. The list should be empty — you will populate it in Module 2.
5. Click **Catalog** under Virtualization. Browse the available boot source templates to familiarize yourself with what images are pre-staged in the cluster.
6. Open a terminal in the showroom environment (the embedded terminal tab). Run `oc whoami` to confirm you are logged in with your assigned user identity.
7. Run `oc project` to confirm your current project. Switch to your dedicated namespace:
   ```
   oc project <your-namespace>
   ```
8. Verify `virtctl` is available:
   ```
   virtctl version
   ```
   Confirm the client version is compatible with the server version reported.
9. Run the following to confirm that the VirtualMachine CRD is present in the cluster:
   ```
   oc get crd virtualmachines.kubevirt.io
   ```
10. Run a quick health check on the Virtualization operator:
    ```
    oc get hyperconverged -n openshift-cnv kubevirt-hyperconverged -o jsonpath='{.status.conditions[?(@.type=="Available")].status}'
    ```
    Confirm the output is `True`.
11. Return to the web console. Click your username in the top-right corner and note the **Copy login command** option — this is the standard way to refresh CLI credentials for the session.

### Key Takeaways

- OpenShift 5 places Virtualization as a first-class top-level navigation item in the console, reflecting its parity with containerized workloads.
- The HyperConverged CR is the single control point for OpenShift Virtualization configuration; its `Available` condition is the authoritative health indicator.
- `virtctl` is a purpose-built CLI for VM-specific operations (console access, start/stop, live migration) that complements the standard `oc` tool.
- Every participant works in an isolated namespace with pre-staged boot images, eliminating environment setup overhead.
