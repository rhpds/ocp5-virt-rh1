# Module 08 — Working with VMs and Applications

### Brief Overview

This module bridges the gap between virtual machine management and application delivery — a key advantage of running VMs on OpenShift rather than a traditional hypervisor. Participants deploy a simple web application inside a VM, expose it using a Kubernetes Service, and make it externally accessible via a Route — the same pattern used for containerized applications on OpenShift. The module demonstrates that VM-hosted workloads are first-class citizens in the OpenShift service mesh and ingress ecosystem. OpenShift 5 further simplifies this integration with improved `virtctl expose` support and direct integration between the VM workload type and the Developer console's topology view.

### Audience and Time

- **Personas:** Platform engineers, application owners managing VM-hosted workloads, infrastructure architects evaluating VM-to-container migration paths
- **Prerequisites for this module:** Module 02 completed; a running VM (`fedora-lab`) in the participant namespace with a web server installable via dnf; cluster ingress and DNS configured
- **Estimated duration:** 20 minutes

### Learning Objectives

- Deploy a simple web application inside a virtual machine and verify it is serving on a local port
- Expose the VM-hosted application using a Kubernetes Service with `virtctl expose`
- Create an OpenShift Route to make the application accessible via the cluster's ingress DNS
- Verify end-to-end application access from outside the cluster using `curl`
- Observe the VM workload in the OpenShift 5 Developer console topology view alongside containerized applications

### Lab Structure

| Section | Title | Duration |
|---------|-------|----------|
| 1 | Deploying an Application Inside the VM | 5 min |
| 2 | Exposing the VM with a Kubernetes Service | 6 min |
| 3 | Creating a Route and Testing External Access | 5 min |
| 4 | Observing VMs in the Developer Console | 4 min |

### Detailed Steps

**Section 1 — Deploying an Application Inside the VM**

1. Ensure `fedora-lab` is running. Open the VM console from the web console or connect via `virtctl`:
   ```
   virtctl ssh fedora@fedora-lab -n <your-namespace>
   ```
2. Install a simple HTTP server inside the VM:
   ```
   sudo dnf install -y httpd
   sudo systemctl enable --now httpd
   ```
3. Create a simple test page to identify responses from this VM:
   ```
   echo "<h1>Hello from $(hostname) on OpenShift 5!</h1>" | sudo tee /var/www/html/index.html
   ```
4. Verify the web server is listening locally:
   ```
   curl http://localhost/
   ```
   Confirm the response includes the hostname.
5. Open the firewall to allow HTTP traffic within the VM (required for the Kubernetes Service port-forward to reach the guest):
   ```
   sudo firewall-cmd --permanent --add-service=http
   sudo firewall-cmd --reload
   ```
6. Exit the SSH session:
   ```
   exit
   ```

**Section 2 — Exposing the VM with a Kubernetes Service**

7. Use `virtctl expose` to create a Kubernetes Service that forwards traffic to port 80 of the VM:
   ```
   virtctl expose vm fedora-lab \
     --name fedora-lab-web \
     --port 80 \
     --target-port 80 \
     --type ClusterIP \
     -n <your-namespace>
   ```
   In OCP5, `virtctl expose` creates a Service that uses the VM's pod as the endpoint backend, just as `kubectl expose` would for a regular pod.
8. Confirm the Service was created:
   ```
   oc get svc fedora-lab-web -n <your-namespace>
   ```
   Note the ClusterIP assigned. The Service selector targets the virt-launcher pod associated with the VM.
9. Verify the Service endpoints are populated (confirming the VM pod is backing the Service):
   ```
   oc get endpoints fedora-lab-web -n <your-namespace>
   ```
   The endpoint IP should match the virt-launcher pod IP.
10. Test connectivity to the Service from within the cluster using a temporary pod:
    ```
    oc run curl-test --image=registry.access.redhat.com/ubi9/ubi-minimal --restart=Never \
      --rm -it -- curl http://fedora-lab-web.<your-namespace>.svc.cluster.local/
    ```
    Confirm the HTML response is returned.

**Section 3 — Creating a Route and Testing External Access**

11. Expose the ClusterIP Service externally using an OpenShift Route:
    ```
    oc expose svc fedora-lab-web \
      --name fedora-lab-route \
      -n <your-namespace>
    ```
12. Retrieve the Route hostname:
    ```
    oc get route fedora-lab-route -n <your-namespace> -o jsonpath='{.spec.host}'
    ```
    The hostname follows the pattern `fedora-lab-web-<namespace>.<cluster-ingress-domain>`.
13. Test external access from the lab terminal (outside the cluster network):
    ```
    curl http://$(oc get route fedora-lab-route -n <your-namespace> -o jsonpath='{.spec.host}')/
    ```
    Confirm the response: `<h1>Hello from fedora-lab on OpenShift 5!</h1>`.
14. Open the Route URL in the showroom browser tab to confirm the application is accessible via a standard web browser.
15. (Optional) Create a TLS-terminated Route to expose the application over HTTPS using the cluster's default wildcard certificate:
    ```
    oc create route edge fedora-lab-secure \
      --service=fedora-lab-web \
      --insecure-policy=Redirect \
      -n <your-namespace>
    ```
    Test HTTPS access:
    ```
    curl https://$(oc get route fedora-lab-secure -n <your-namespace> -o jsonpath='{.spec.host}')/
    ```

**Section 4 — Observing VMs in the Developer Console**

16. Switch to the **Developer** perspective in the OpenShift 5 console (click the perspective dropdown at the top of the left sidebar and select **Developer**).
17. Navigate to **Topology**. Confirm that `fedora-lab` appears in the topology graph as a VM workload node alongside any containerized applications in your namespace.
18. In OCP5, VM nodes in the Topology view display the VM status badge (Running/Stopped), the associated Service and Route as connected arrows, and a right-click context menu with VM-specific actions (start, stop, migrate).
19. Click the `fedora-lab` node. The side panel shows the VM overview, the URL of the Route, and resource metrics. Click the Route URL to open the application in a new browser tab.
20. Return to the **Administrator** perspective. Inspect the relationship between the VM, Service, and Route:
    ```
    oc get vm,svc,route -n <your-namespace>
    ```

### Key Takeaways

- VM-hosted applications on OpenShift are exposed using the same Kubernetes Service and Route primitives as containerized applications — no special networking configuration is required.
- `virtctl expose` creates a standard Kubernetes Service backed by the VM's pod endpoint, making the VM's application reachable within the cluster immediately.
- OpenShift Routes provide external HTTP/HTTPS access via the cluster ingress controller, including TLS termination with the cluster's wildcard certificate — without requiring a load balancer per VM.
- OpenShift 5's Developer console topology view integrates VM workloads alongside pods and deployments, giving application teams a unified view of all workload types in a namespace.
- The ability to expose VM-hosted applications through standard OpenShift networking is a key architectural advantage for workloads that cannot be immediately containerized but still need to integrate with container-native services.
