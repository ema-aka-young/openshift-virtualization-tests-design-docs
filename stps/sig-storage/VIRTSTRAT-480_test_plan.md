# Openshift-virtualization-tests Test plan

## **File-Level Restore for Data Protection Partners - Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** [VEP: File-Level Restore](https://github.com/kubevirt/enhancements/pull/170)
- **Feature Tracking:** [VIRTSTRAT-480](https://issues.redhat.com/browse/VIRTSTRAT-480) - support single file restore for data protection partners
- **Epic Tracking:** [CNV-73895](https://issues.redhat.com/browse/CNV-73895) - Dev Preview: File-Level Restore
- **Feature Maturity:**
  - DP: 5.0 (CNV-73895)
  - TP: 5.2 (CNV-89069)
  - GA: 5.2 (CNV-89094)
- **QE Owner(s):** Emanuele Prella
- **Owning SIG:** sig-storage
- **Participating SIGs:** sig-compute (guest OS detection, SSH access), sig-network (VM IP resolution, VSOCK)
- **Child STPs:** N/A
- **Document Conventions (if applicable):** N/A

### **Feature Overview**

This feature introduces file-level restore capabilities for VMs running on OpenShift Virtualization, allowing users and data protection partners to restore specific files or directories from a volume snapshot or a backup PVC into a running VM, without requiring a full volume or VM restore. This brings parity with file-level restore functionality available on traditional virtualization platforms today.

The implementation is delivered through a new standalone operator (vm-file-restore-operator) that introduces the `VirtualMachineFileRestore` CRD (`filerestore.kubevirt.io/v1alpha1`). The operator supports two restore modes: automatic (with `sourcePath` -- copies files via `rsync`) and manual (without `sourcePath` -- mounts backup read-only for user browsing). Two source types are supported: backup PVC and VolumeSnapshot. Guest OS support covers Linux (via `filerestore.sh` helper) and Windows (via `filerestore.bat` helper), with the operator using SSH with command-restricted keys for secure guest command execution.

This STP covers the Dev Preview (DP) phase targeting CNV v5.0.0. The test scope focuses on core functional validation of the CRD lifecycle, restore workflows, and integration with existing KubeVirt features (declarative hotplug volumes, VM snapshots, CDI DataVolumes). Encryption support (LUKS/BitLocker) and API extensions are tracked under CNV-89229 for v5.1.0.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

This section documents the mandatory QE review process. The goal is to understand the feature's value,
technology, and testability before formal test planning.

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:*
    - [CNV-73895] Implement file-level restore for VMs, allowing restore of specific files or directories from a volume snapshot or a backup PVC into a running VM
    - [CNV-88321] Develop HCO-compliant operator that deploys and manages the file restore controller with CSV generator, certificate rotation, TLS configuration, and operator-sdk bundle generation
    - [CNV-88322] File restore controller with basic functionality implementing phase-based reconciliation (WaitingForAttachment, Connecting, Restoring, Cleanup, Succeeded/Failed)
    - [CNV-88323] Linux guest file restore helper that detects hotplug disk, mounts, copies files, and reports status
    - [CNV-88324] Windows guest file restore helper that detects hotplug disk, mounts, copies files, and reports status
    - [CNV-90086] Set source volume mode from snapcontent.sourceVolumeMode to prevent VolumeMode mismatch failures when restoring from VolumeSnapshot

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* VM users regularly need to partially restore information from a VM backup. Currently, they need to restore an entire VM separately and retrieve the data from it. This feature allows users to find and restore only the specific files needed, significantly reducing restore time and complexity. Data protection partners can offer the same file-level restore capability they have on traditional virtualization platforms today.
  - *List the customer use cases identified:*
    - As a backup vendor, I want to trigger file-level restore from a backup PVC I maintain, so I can offer file level restore capability to my customers
    - As a VM Admin/User, I want to restore specific files or directories from a volume snapshot into my running VM, so I don't have to restore the entire volume or VM
    - As a VM Admin, I want to mount a backup volume read-only to browse and manually copy files without requiring an automated restore path

- [x] **Testability**
  - *Note any requirements that are unclear or untestable:*
    - Guest provisioning requirements (filerestore user, SSH keys, helper scripts) must be pre-installed in the guest. The mechanism for initial guest setup is not fully specified and may vary by VM template.
    - NTFS metadata preservation guarantees (ACLs, alternate data streams, hard links, junction points) are documented as known limitations in the VEP but not as explicit acceptance criteria.

- [x] **Acceptance Criteria**
  - *List the acceptance criteria:*
    - AC-1: Backup vendor can trigger file-level restore from a backup PVC and files are restored to the running VM
    - AC-2: VM Admin/User can restore specific files or directories from a volume snapshot into a running VM without full volume restore
    - AC-3: Linux guest file restore helper detects hotplug disk, mounts, copies files, and reports status
    - AC-4: Windows guest file restore helper detects hotplug disk, mounts, copies files, and reports status
    - AC-5: HCO-compliant operator deploys and manages the file restore controller
    - AC-6: VolumeSnapshot source restores handle VolumeMode correctly from snapcontent.sourceVolumeMode
    - AC-7: Manual restore mode mounts backup filesystem read-only and keeps it available until CR is deleted
    - AC-8: Operator reports clear error conditions when restore fails (missing helper, SSH failure, disk not found)
  - *Note any gaps or missing criteria:*
    - Formal acceptance criteria are not explicitly defined in VIRTSTRAT-480 (feature-level). Criteria above are derived from child epics and user stories.
    - No explicit performance targets for restore operations (e.g., maximum time for 1GB file restore).
    - LUKS/BitLocker encrypted volume restore criteria are deferred to CNV-89229 (v5.1.0).

- [x] **Non-Functional Requirements (NFRs)**
  - *List applicable NFRs and their targets:*
    - Security: Guest command execution restricted to allowed binaries with sanitized arguments via command-restricted SSH keys
    - Security: SSH key lifecycle owned by standalone operator (generated per-restore, cleaned up after completion)
    - Usability: VirtualMachineFileRestore CR status conditions provide clear feedback (Progressing, Completed) with human-readable messages
    - Monitoring: Operator should expose metrics for restore duration, success/failure counts (not yet specified)
  - *Note any NFRs not covered and why:*
    - Performance benchmarking: No formal latency or throughput targets defined for Dev Preview
    - Scale testing: Not applicable for Dev Preview (single-VM restore only, concurrent multi-VM is best-effort)
    - Portability: Operator is Kubernetes-native, no cloud-specific requirements

#### **2. Known Limitations**

The limitations are documented to ensure alignment between development, QA, and product teams.
The following are confirmed product constraints accepted before testing begins.

- Backup file browsing is NOT supported -- users must know the path to restore. However, manual mode allows mounting the backup volume read-only for browsing.
- Parallel file restores of the same VM are NOT supported -- a second VirtualMachineFileRestore CR targeting the same VM will be rejected while one is in progress.
- Remote storage source (S3/rclone) is NOT yet implemented -- the API type exists but returns "remote sources not yet supported" error.
- Guest must have filerestore user, SSH access, and helper scripts pre-installed -- the operator does not provision the guest.
- SSH over VSOCK is not implemented in the current version -- the operator uses standard TCP SSH (port 22).
- NTFS metadata preservation (ACLs, alternate data streams, hard links, junction points) is not guaranteed -- the Windows helper uses basic file copy.
- LVM snapshot UUID collisions are a known issue under investigation (CNV-84209) -- restoring from LVM-based volume snapshots may fail due to duplicate filesystem UUIDs.
- Cross-namespace PVC sources are not supported -- backup PVC must be in the same namespace as the VirtualMachine.
- The DeclarativeHotplugVolumes feature gate must be enabled and the legacy HotplugVolumes gate must NOT be simultaneously enabled (mutual exclusion).

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:*
    - VEP (design doc) merged at https://github.com/kubevirt/enhancements/pull/170 with feedback from 3 data protection partners (Backup/DR Vendor A, Backup/DR Vendor B, and one anonymous via RH Engineering Partner Manager)
    - Architecture decision: standalone operator instead of embedding in KubeVirt core -- keeps KubeVirt at infrastructure layer
    - Phase-based reconciliation loop: WaitingForAttachment -> Connecting -> Restoring -> Cleanup -> Succeeded/Failed
    - SSH with ED25519 command-restricted keys for secure guest access (not using qemu-guest-agent due to security concerns with confidential guests)
    - PoC demonstrated in Jan 15 Sprint demo (CNV-73842)

- [x] **Technology Challenges**
  - *List identified challenges:*
    - Guest OS detection reliability: Cannot rely solely on VM template annotations; must also check GuestOSInfo from guest agent
    - SSH connectivity to Windows guests: Windows SSH server configuration is more complex than Linux
    - PVC size determination: When creating DataVolume from VolumeSnapshot, minimal size from storage class may be insufficient
    - Multi-partition backup PVCs: Need sourcePartition support for backup disks with multiple partitions
    - Disk signature/UUID collisions: LVM-based snapshots may produce duplicate UUIDs that need special handling
  - *Impact on testing approach:* Tests must verify error handling for all edge cases (missing helpers, SSH failures, incorrect OS detection). Windows tests require special environment setup.

- [x] **API Extensions**
  - *List new or modified APIs:*
    - NEW: `filerestore.kubevirt.io/v1alpha1/VirtualMachineFileRestore` CRD -- namespace-scoped, created in VirtualMachine namespace
    - Fields: spec.target (VM reference), spec.source.pvc (PVC source), spec.source.snapshot (VolumeSnapshot source), spec.sourcePath (path to restore), spec.sourcePartition (optional partition number), spec.targetPath (optional target override)
    - Status conditions: Completed, Progressing
    - No modifications to existing KubeVirt APIs -- operator is a consumer of existing declarative hotplug volumes API
  - *Testing impact:* All CRD lifecycle operations (create, reconcile, delete) must be tested. RBAC roles (admin, editor, viewer) should be validated.

- [x] **Test Environment Needs**
  - *See environment requirements in Section II.3 and testing tools in Section II.3.1*

- [x] **Topology Considerations**
  - *Describe topology requirements:* Standard multi-node cluster with at least 2 worker nodes (for live migration tests). No multi-cluster requirements.
  - *Impact on test design:* Live migration tests with restore volume require schedulable target node. Windows VM tests require sufficient memory allocation.

### **II. Software Test Plan (STP)**

This STP serves as the **overall roadmap for testing**, detailing the scope, approach, resources, and schedule.

#### **1. Scope of Testing**

Testing covers the Dev Preview (CNV v5.0.0) of the file-level restore feature delivered through the vm-file-restore-operator. The scope includes:

- VirtualMachineFileRestore CRD lifecycle: create, progress through status phases, cleanup on deletion
- Automatic restore mode: file copy from backup source to running VM via rsync and guest helper
- Manual restore mode: read-only mount of backup volume for user browsing
- Restore sources: backup PVC (same namespace) and VolumeSnapshot (converted to DataVolume)
- Guest OS support: Linux (RHEL) and Windows Server 2022
- Integration with declarative hotplug volumes (Hotpluggable=true, SCSI bus, VolumeReady phase)
- Integration with CDI DataVolumes (DataVolumeSourceSnapshot)
- SSH connectivity with command-restricted keys
- Error handling and validation (cross-namespace rejection, concurrent restore rejection, missing helpers)
- Regression testing for hotplug-dependent features (VM snapshots, live migration, CBT)

**Testing Goals**

- [P0] Verify automatic file restore from VolumeSnapshot source completes successfully on Linux VM and restores files with original content
- [P0] Verify automatic file restore from backup PVC source completes successfully on Linux VM
- [P0] Verify manual file restore mounts backup volume read-only and files are accessible at backup mount path
- [P0] Verify VMFR CR cleanup (deletion) removes hotplug volume from VM and cleans up DataVolume
- [P1] Verify declarative hotplug volume attaches with SCSI bus, reaches VolumeReady phase, and detaches correctly
- [P1] Verify VolumeSnapshot-to-DataVolume conversion produces a PVC that can be successfully hotplugged
- [P1] Verify SSH connection with command-restricted key succeeds for restore operations and rejects arbitrary commands
- [P1] Verify file restore from secondary data disk snapshot succeeds
- [P1] Verify VM snapshot workflow is not disrupted when a restore hotplug volume is present
- [P2] Verify Windows guest file restore in both automatic and manual modes
- [P2] Verify sequential file restores from the same snapshot all succeed
- [P2] Verify large file (1GB) restore completes within 10 minutes
- [P2] Verify live migration completes with restore hotplug volume attached
- [P2] Verify concurrent restore on same VM is rejected with clear error
- [P2] Verify CBT backup operations are not disrupted by file restore hotplug lifecycle

**Out of Scope (Testing Scope Exclusions)**

- Remote storage (S3/rclone) sources -- API type exists but not implemented; explicitly returns "not supported" error
- Encrypted volume restore (LUKS/BitLocker) -- deferred to CNV-89229 (v5.1.0)
- Performance benchmarking -- no formal targets defined for Dev Preview
- Scale testing beyond basic multi-VM concurrency (2 VMs)
- Backup vendor proprietary format testing -- out of scope for platform-level QE
- Guest provisioning automation -- operator requires pre-provisioned guest (filerestore user, SSH keys, helper scripts)
- ARM64 architecture -- not targeted for Dev Preview

**Test Limitations**

- Windows SSH server configuration requires manual setup and may vary by Windows Server version
- LVM snapshot UUID collision scenarios (CNV-84209) are under investigation and may not be testable until the spike is completed
- Multi-partition backup PVCs require specific storage configurations that may not be available in all test environments
- Guest helper script testing (filerestore.sh, filerestore.bat) depends on guest-internal state that is difficult to mock

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** -- Validates that the feature works according to specified requirements and user stories
  - *Details:* Core functional testing covers all CRD lifecycle operations (create, reconcile, delete), both restore modes (automatic/manual), both source types (PVC/VolumeSnapshot), and both guest OS types (Linux/Windows). Tests verify that files are correctly restored with original content and that cleanup properly removes temporary resources.

- [x] **Automation Testing** -- Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)
  - *Details:* Tier 1 tests (Go/Ginkgo) target kubevirt/kubevirt integration points (hotplug volumes, VolumeSnapshot-to-DataVolume, admission). Tier 2 tests (Python/pytest) target end-to-end workflows in openshift-virtualization-tests/tests/storage/file_level_restore/. Existing tier 2 test framework already contains test fixtures for operator installation, VM provisioning, and file restore operations. All tests will be integrated into the nightly CI lane for sig-storage.

- [x] **Regression Testing** -- Verifies that new changes do not break existing functionality
  - *Details:* LSP-based regression analysis identified 9 impacted features through call graph tracing. Key regression areas: declarative hotplug volumes (HandleDeclarativeVolumes), VolumeReady phase status reporting, VM snapshot with hotplug volumes, live migration with hotplug volumes, and CBT backup operations. Regression scenarios are included in Section III as Tier 1 and Tier 3 tests.

**Non-Functional**

- [ ] **Performance Testing** -- Validates feature performance meets requirements (latency, throughput, resource usage)
  - *Details:* N/A for Dev Preview. Large file restore test (1GB) provides basic performance validation. Formal benchmarking deferred to TP/GA phases.

- [ ] **Scale Testing** -- Validates feature behavior under increased load and at production-like scale
  - *Details:* N/A for Dev Preview. Basic concurrent multi-VM test (2 VMs) included. Full scale testing deferred to TP/GA phases.

- [x] **Security Testing** -- Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - *Details:* Verify SSH command-restricted keys reject arbitrary command execution. Verify RBAC roles (admin, editor, viewer) enforce correct access to VirtualMachineFileRestore resources. Verify cross-namespace PVC access is rejected.

- [x] **Usability Testing** -- Validates user experience and accessibility requirements
  - *Details:* No UI component — usability refers to API/CLI status reporting quality. Validate that VirtualMachineFileRestore status conditions provide clear feedback about operation progress and outcome (Progressing, Completed conditions with human-readable messages). Validate that error conditions are informative (missing helper, SSH failure, concurrent restore rejection).

- [ ] **Monitoring** -- Does the feature require metrics and/or alerts?
  - *Details:* Monitoring requirements not yet specified for Dev Preview. Operator may expose basic Prometheus metrics in future versions.

**Integration & Compatibility**

- [x] **Compatibility Testing** -- Ensures feature works across supported platforms, versions, and configurations
  - *Details:* Feature requires DeclarativeHotplugVolumes feature gate (new in recent KubeVirt versions). Backward compatibility is N/A as this is a new CRD/operator. Must verify the feature gate mutual exclusion with legacy HotplugVolumes gate.

- [x] **Upgrade Testing** -- Validates upgrade paths from previous versions, data migration, and configuration preservation
  - *Details:* As a new operator, upgrade testing focuses on: operator upgrade preserving in-progress restores (if any), CRD schema versioning (v1alpha1), and HCO integration during cluster upgrade.

- [x] **Dependencies** -- Blocked by deliverables from other components/products
  - *Details:*
    - Blocked on DeclarativeHotplugVolumes feature gate availability in KubeVirt CR
    - Blocked on CDI support for DataVolumeSourceSnapshot with correct VolumeMode handling
    - Blocked on vm-file-restore-operator downstream build availability (Konflux builds -- CNV-89642)

- [x] **Cross Integrations** -- Does the feature affect other features or require testing by other teams?
  - *Details:*
    - Hotplug volume lifecycle may interact with CBT (Changed Block Tracking) backup operations -- verify CBT checkpoint tracking handles restore volumes
    - Hotplug volume presence during live migration -- verify migration succeeds with restore volume attached
    - VM snapshot workflow with hotplug volumes -- verify online snapshot succeeds with restore volume present
    - VSOCK integration (CNV-64172, CNV-76488) -- future integration point, not tested in DP phase

**Infrastructure**

- [ ] **Cloud Testing** -- Does the feature require multi-cloud platform testing?
  - *Details:* N/A. Feature is storage-provider agnostic. Standard OCP cluster with CSI VolumeSnapshot support is sufficient.

#### **3. Test Environment**

- **Cluster Topology:** Multi-node (minimum 2 worker nodes for live migration tests)

- **OCP & OpenShift Virtualization Version(s):** OCP 4.22+ / CNV 5.0.0+ (Dev Preview)

- **CPU Virtualization:** Standard (no specific CPU requirements)

- **Compute Resources:** Minimum 2 worker nodes with 16GB RAM each (Windows VM tests require 8GB per VM)

- **Special Hardware:** None required

- **Storage:** ocs-storagecluster-ceph-rbd with VolumeSnapshot CSI support (pre-existing infrastructure provided by OpenShift Data Foundation). Must support both Block and Filesystem VolumeMode.

- **Network:** Standard OVN-Kubernetes CNI. No secondary network requirements.

- **Required Operators:**
  - OpenShift Virtualization (kubevirt-hyperconverged) with DeclarativeHotplugVolumes feature gate enabled
  - vm-file-restore-operator (new operator under test)
  - OpenShift Data Foundation (for CSI VolumeSnapshot support)

- **Platform:** Any supported OCP platform (bare-metal or supported cloud providers)

- **Special Configurations:** DeclarativeHotplugVolumes feature gate must be enabled in KubeVirt CR. Legacy HotplugVolumes feature gate must NOT be simultaneously enabled.

#### **3.1. Testing Tools & Frameworks**

Standard project testing infrastructure (Go/Ginkgo v2, Python/pytest, virtctl, oc). Additional tool:

- `ssh` — Required for guest OS verification of restored files and command-restricted key validation

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [x] Requirements and design documents are **approved and merged** (VEP merged: https://github.com/kubevirt/enhancements/pull/170)
- [ ] Test environment can be **set up and configured** (see Section II.3 - Test Environment)
- [ ] vm-file-restore-operator is built and available for deployment on test cluster
- [ ] DeclarativeHotplugVolumes feature gate is available and can be enabled in KubeVirt CR
- [ ] Guest VM templates include filerestore user, SSH access, and helper scripts (or provisioning process is documented)
- [ ] Downstream builds of operator containers are available via Konflux (CNV-89642)

#### **5. Risks**

**Timeline/Schedule**

- **Risk:** Feature development is still in progress (CNV-88320, CNV-88321, CNV-88322, CNV-88323, CNV-88324 are In Progress or New). Test development may be blocked by incomplete implementation.
  - **Mitigation:** QE spike (CNV-86827) already completed. Existing tier 2 test framework can be developed against PoC. Prioritize tier 1 integration tests that use existing KubeVirt APIs.
  - *Estimated impact on schedule:* 2-3 week delay possible if operator is not ready for integration testing
  - *Sign-off:* Pending

**Test Coverage**

- **Risk:** Guest provisioning story is incomplete -- how filerestore user, SSH keys, and helper scripts are installed in VMs is not fully specified. This may prevent automated test execution.
  - **Mitigation:** Work with development team to document guest provisioning requirements. Use cloud-init or Ignition to provision test VMs. Create reusable test fixtures for guest setup.
  - *Areas with reduced coverage:* Guest helper edge cases (corrupted helper binary, partial installation)
  - *Sign-off:* Pending

**Test Environment**

- **Risk:** DeclarativeHotplugVolumes feature gate may not be available in the test cluster version, or may conflict with existing tests that use legacy HotplugVolumes gate.
  - **Mitigation:** Verify feature gate availability early. Coordinate with CI team to ensure test lane uses correct feature gate configuration. Use separate test profile if needed.
  - *Missing resources or infrastructure:* Feature gate configuration in test cluster
  - *Sign-off:* Pending

**Untestable Aspects**

- **Risk:** NTFS metadata preservation (ACLs, alternate data streams) cannot be fully validated in automated tests.
  - **Mitigation:** Document as known limitation. Perform manual verification for critical NTFS metadata scenarios.
  - *Alternative validation approach:* Manual verification with Windows-specific tools (icacls, streams.exe)
  - *Sign-off:* Pending

**Resource Constraints**

- **Risk:** Windows VM tests require significant compute resources (8GB RAM per VM) and special setup.
  - **Mitigation:** Run Windows tests in dedicated test lane with sufficient resources. Limit Windows tests to Tier 3 priority.
  - *Current capacity gaps:* Windows test infrastructure availability
  - *Sign-off:* Pending

**Dependencies**

- **Risk:** Downstream builds of vm-file-restore-operator containers may not be available in time for test development. Konflux build setup (CNV-89642) is still in To Do status.
  - **Mitigation:** Use upstream operator images for initial test development. Switch to downstream images once Konflux builds are available.
  - *Dependent teams or components:* Release Engineering (Konflux builds), Development (operator stability), CDI team (DataVolume from snapshot), Storage team (VolumeSnapshot CSI support)
  - *Sign-off:* Pending

**Other**

- **Risk:** LVM snapshot UUID collision issue (CNV-84209) may cause test failures for LVM-based storage providers. The spike is still in To Do status.
  - **Mitigation:** Skip LVM-specific tests until spike is completed. Document as known limitation. Use non-LVM storage class for initial testing.
  - *Sign-off:* Pending

---

### **III. Test Scenarios & Traceability**

#### Tier 1 -- Functional Tests (Go/Ginkgo)

- **[CNV-73895]** -- Declarative hotplug volume attach and detach for file restore
  - Verify that a PVC volume with `Hotpluggable=true` and SCSI disk bus can be declaratively hotplugged to a running VM, that the VM reports the volume as ready and attached, and that removing the volume from the VM spec triggers detachment.
  - **Priority:** P1
  - **Polarion:** Yes
  - **LSP Evidence:** `HandleDeclarativeVolumes` at `kubevirt/pkg/storage/hotplug/hotplug.go:37`

- **[CNV-73895, CNV-90086]** -- VolumeSnapshot-to-DataVolume produces usable PVC for hotplug
  - Verify that creating a DataVolume from a VolumeSnapshot source (using `DataVolumeSourceSnapshot`) results in a PVC that can be hotplugged to a running VM with correct VolumeMode inherited from the snapshot source.
  - **Priority:** P1
  - **Polarion:** Yes
  - **LSP Evidence:** `DataVolumeSourceSnapshot` at `kubevirt/pkg/storage/types/dv.go`

- **[CNV-88323, CNV-88324]** -- SSH connection with command-restricted key
  - Verify that the file restore operator's SSH connection using an ED25519 command-restricted key successfully connects to the guest as the `filerestore` user, that the restricted command (guest helper) can be executed, and that arbitrary commands are rejected by the SSH key restriction.
  - **Priority:** P1
  - **Polarion:** Yes
  - **LSP Evidence:** `ConnectSSH` at `vm-file-restore-operator/internal/controller/ssh.go:22`

- **[CNV-73895]** -- VMFR CR cleanup removes hotplug volume on deletion
  - Verify that deleting a `VirtualMachineFileRestore` CR triggers the cleanup finalizer, which removes the hotplug volume from the VM spec and deletes the associated DataVolume (if the source was a VolumeSnapshot).
  - **Priority:** P0
  - **Polarion:** Yes
  - **LSP Evidence:** Controller cleanup at `virtualmachinefilerestore_controller.go:153`

- **[CNV-73895]** -- Hotplug volume does not break existing VM snapshot workflow
  - Verify that taking an online VM snapshot while a restore hotplug volume (declarative, SCSI, `Hotpluggable=true`) is attached to the VM succeeds and the snapshot includes the hotplug volume data.
  - **Priority:** P1
  - **Polarion:** Yes
  - **LSP Evidence:** VM snapshot tests at `tests/storage/snapshot.go:569`

- **[CNV-88323, CNV-88324]** -- Guest OS detection identifies Linux and Windows VMs
  - Verify that the operator correctly identifies the guest OS type and uses the correct helper script paths and mount paths for each OS type.
  - **Priority:** P2
  - **Polarion:** Yes
  - **LSP Evidence:** `DetectGuestOS` at `vm-file-restore-operator/internal/controller/os.go:28`

- **[CNV-73895]** -- Live migration with hotplug restore volume succeeds
  - Verify that a VM with a declarative hotplug restore volume (SCSI bus, `Hotpluggable=true`) can be live migrated successfully, and that the volume remains attached and accessible in the guest after migration.
  - **Priority:** P2
  - **Polarion:** Yes
  - **LSP Evidence:** `GetHotplugVolumes` at `kubevirt/pkg/storage/types/volume.go:77` called by migration controller

- **[CNV-73895]** -- Concurrent restore on same VM is rejected
  - Verify that creating a second `VirtualMachineFileRestore` CR targeting a VM that already has a restore in progress results in an error condition indicating another restore is in progress.
  - **Priority:** P2
  - **Polarion:** Yes
  - **LSP Evidence:** `hotplug.go:38-40` checks for existing `-restore` volumes

- **[CNV-73895]** -- Cross-namespace PVC restore is rejected
  - Verify that creating a `VirtualMachineFileRestore` CR with a PVC source referencing a PVC in a different namespace than the target VM results in a validation error with an appropriate message.
  - **Priority:** P3
  - **Polarion:** Yes
  - **LSP Evidence:** `hotplug.go:80-82` validates cross-namespace PVC references

#### Tier 2 -- End-to-End Tests (Python/pytest)

- **[CNV-73895]** -- Automatic file restore from VolumeSnapshot on Linux VM
  - Install the file-restore operator, create a RHEL VM with the filerestore user configured, write a test file with known content, take a VolumeSnapshot of the root disk, delete the test file, create a `VirtualMachineFileRestore` CR with `sourcePath` set to the test file path, wait for the CR to reach Succeeded phase, and verify the file is restored with its original content.
  - **Priority:** P0
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestore::test_automatic_file_restore_from_snapshot`

- **[CNV-73895]** -- Manual file restore from backup PVC on Linux VM
  - Install the file-restore operator, create a RHEL VM, write a test file, create a backup PVC containing the file, create a `VirtualMachineFileRestore` CR without `sourcePath` (manual mode), wait for VolumeReady phase, and verify the backup volume is mounted read-only and the file is accessible at the backup mount path inside the guest.
  - **Priority:** P0
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestore::test_manual_file_restore_from_pvc`

- **[CNV-73895]** -- Automatic file restore from backup PVC on Linux VM
  - Install the file-restore operator, create a RHEL VM, write a test file, create a backup PVC containing the file, delete the test file from the VM, create a `VirtualMachineFileRestore` CR with `sourcePath` set to the test file path and PVC source, wait for the CR to reach Succeeded phase, and verify the file is restored with its original content.
  - **Priority:** P0
  - **Polarion:** Yes

- **[CNV-73895]** -- Automatic file restore from data disk snapshot
  - Create a RHEL VM with a secondary data disk (hotplugged DataVolume), format and mount the data disk, write a test file to the data disk, take a VolumeSnapshot of the data disk, delete the test file, create a `VirtualMachineFileRestore` CR with `sourcePath` pointing to the file on the data disk, wait for Succeeded phase, and verify the file is restored.
  - **Priority:** P1
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestoreDataDisk::test_automatic_file_restore_from_data_disk_snapshot`

- **[CNV-73895]** -- Multiple sequential file restores from same snapshot
  - Create a RHEL VM with a data disk, write 3 test files, take a VolumeSnapshot, delete all files, create 3 sequential `VirtualMachineFileRestore` CRs (waiting for each to complete before creating the next), and verify all 3 files are restored with their original content.
  - **Priority:** P2
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestoreDataDisk::test_sequential_automatic_file_restores_from_data_disk_snapshot`

- **[CNV-73895]** -- Large file (1GB) restore from data disk snapshot
  - Create a RHEL VM with a data disk, write a 1GB test file using `dd`, take a VolumeSnapshot, delete the file, create a `VirtualMachineFileRestore` CR with `sourcePath`, wait for Succeeded phase with a 10-minute timeout, and verify the file is restored with the correct size (1GB).
  - **Priority:** P2
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestoreDataDisk::test_automatic_big_file_restore_from_data_disk_snapshot`

#### Tier 3 -- Specialized Tests (Python/pytest)

- **[CNV-88324]** -- Windows guest file restore (automatic and manual modes)
  - Install the file-restore operator, create a Windows Server 2022 VM with the filerestore user and SSH key configured, write a test file. For manual mode: create a `VirtualMachineFileRestore` CR without `sourcePath`, wait for VolumeReady, verify file accessible at backup junction path. For automatic mode: delete the file, create a CR with `sourcePath`, wait for Succeeded, verify file restored.
  - **Priority:** P2
  - **Polarion:** Yes
  - **Existing Tests:** `test_file_level_restore.py::TestFileRestoreWindows`

- **[CNV-73895]** -- Concurrent file restore on multiple VMs
  - Create two VMs (one with root disk restore, one with data disk restore), prepare test files and VolumeSnapshots on both VMs, create two `VirtualMachineFileRestore` CRs simultaneously targeting different VMs, and verify both restores complete successfully.
  - **Priority:** P2
  - **Polarion:** Yes
  - **Existing Test:** `test_file_level_restore.py::TestFileRestoreConcurrentMultiVM`

- **[CNV-73895]** -- CBT backup not disrupted by file restore hotplug operations
  - Enable CBT (Changed Block Tracking) on a VM, perform a full backup, hotplug a restore volume via the file restore operator, verify that CBT checkpoint tracking correctly handles the new volume, remove the restore volume, and verify that an incremental backup still works correctly.
  - **Priority:** P2
  - **Polarion:** Yes
  - **LSP Evidence:** CBT at `kubevirt/pkg/storage/cbt/push-target-pvc.go:93`

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - Emanuele Prella
  - Arnon Gilboa (Dev Lead)
* **Approvers:**
  - Natalie Gavrielov (Engineering Manager)
  - QE Lead (sig-storage)
