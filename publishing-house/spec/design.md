# OpenShift 5 Virtualization Lab

## Overview

This lab introduces participants to virtual machine management in Red Hat OpenShift 5, covering both foundational OpenShift Virtualization capabilities and new features introduced in OpenShift 5. It is designed as an updated successor to the OpenShift Virtualization Roadshow, incorporating OCP5-specific workflows and tooling.

Participants will create and manage VMs using the OpenShift console and CLI, configure VM networking and storage, explore new OpenShift 5 virtualization capabilities, and migrate existing VMs using the Migration Toolkit for Virtualization. Modules are self-contained and can be completed independently, allowing participants to focus on areas most relevant to their role or use the lab as a skills refresher.

## Target Audience

- **Role:** Platform engineers, infrastructure architects, virtualization administrators (including VMware and other hypervisor admins evaluating or transitioning to OpenShift)
- **Experience level:** Intermediate
- **What they already know:** Basic Linux and general infrastructure concepts; familiarity with virtualization fundamentals (VMs, hypervisors, networking)
- **What they don't know:** OpenShift Virtualization capabilities and workflows; OpenShift 5-specific virtualization features; VM lifecycle management in a Kubernetes-native environment

## Prerequisites

- Basic familiarity with virtualization concepts (virtual machines, hypervisors, networking)
- General Linux command-line comfort
- No prior OpenShift or Kubernetes experience required

Can the lab validate these automatically? No — trust-based. The lab is structured to be accessible without prior OpenShift experience, with enough detail that any participant can follow along.

## Learning Objectives

1. Create and manage virtual machines using the Red Hat OpenShift Virtualization console and CLI
2. Demonstrate live migration and core VM lifecycle operations in OpenShift 5
3. Configure VM networking within an OpenShift cluster using network attachment definitions and user-defined networks
4. Configure VM storage within an OpenShift cluster using persistent volumes, snapshots, and clones
5. Implement backup and recovery for virtual machines using OADP
6. Deploy and manage VM templates and instance types for standardized VM provisioning
7. Expose virtual machine-hosted applications using OpenShift services and routes
8. Migrate virtual machines from external hypervisors into OpenShift using Migration Toolkit for Virtualization
9. Explore new OpenShift 5 virtualization capabilities integrated throughout the platform

## Content Type

Lab (hands-on)

## Products & Technologies

- Red Hat OpenShift
- Red Hat OpenShift Virtualization
- Red Hat OpenShift Data Foundation
- Migration Toolkit for Virtualization
- OpenShift API for Data Protection (OADP)

## Module Map

| Module | Title | Duration |
|--------|-------|----------|
| 1 | Introduction and Environment Overview | 15 min |
| 2 | Virtual Machine Management | 30 min |
| 3 | Migrating Existing VMs with MTV | 30 min |
| 4 | VM Storage Management | 20 min |
| 5 | Backup and Recovery with OADP | 20 min |
| 6 | Templates and InstanceType Management | 20 min |
| 7 | VM Networking | 25 min |
| 8 | Working with VMs and Applications | 20 min |
| — | **Total hands-on** | **~3 hr** |
| — | Intro / orientation | ~10 min |
| — | **Total lab** | **~3 hr 10 min** |

*OpenShift 5 new features are highlighted throughout each module rather than in a dedicated section.*

## Difficulty Level

Intermediate

## Environment

**Learner view:** Participants access a pre-deployed OpenShift 5 cluster with the OpenShift Virtualization operator and Migration Toolkit for Virtualization already installed. Each participant has a dedicated namespace. Sample VM boot images are pre-staged in the cluster. The OpenShift web console and `oc` / `virtctl` CLI tools are available from the lab environment.

**Automation needed:** Yes

- OpenShift Virtualization operator installed and configured
- OpenShift Data Foundation installed and configured
- Migration Toolkit for Virtualization operator installed
- OADP operator installed and configured
- Per-participant namespace with appropriate RBAC
- Sample VM boot images pre-staged (e.g., RHEL or Fedora disk image available as a DataVolume source)

## Infrastructure Requirements

- **Cloud provider:** TBD — confirmed in infrastructure phase
- **Cluster type:** TBD — confirmed in infrastructure phase
- **OCP version:** TBD — confirmed in infrastructure phase
- **Topology:** TBD — confirmed in infrastructure phase
- **Sizing:** TBD — confirmed in infrastructure phase
- **Automation approach:** TBD — confirmed in infrastructure phase
- **AI/MaaS:** TBD — confirmed in infrastructure phase
- **External services:** TBD — confirmed in infrastructure phase
- **AAP version:** TBD — confirmed in infrastructure phase
- **Non-GA products:** TBD — confirmed in infrastructure phase
