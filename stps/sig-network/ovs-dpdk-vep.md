# Openshift-virtualization-tests Test plan

## **OVS-DPDK Network Binding (via DRA) In Virtual Machines - Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** N/A — scope defined by [CNV-88618](https://redhat.atlassian.net/browse/CNV-88618)
- **Feature Tracking:** https://issues.redhat.com/browse/VIRTSTRAT-639
- **Epic Tracking:** https://redhat.atlassian.net/browse/CNV-88618
- **Feature Maturity:**
  - DP: N/A
  - TP: 5.1
  - GA: 5.2
- **QE Owner(s):** Yossi Segev (ysegev@redhat.com)
- **Owning SIG:** sig-network
- **Participating SIGs:** sig-network

**Document Conventions:**

- DRA: Dynamic Resource Allocation — the Kubernetes-native, claim-based device management framework
- RC: ResourceClaim — a Kubernetes object requesting specific devices for a single workload instance
- RCT: ResourceClaimTemplate — a Kubernetes object that generates a per-VM ResourceClaim, required for live migration of VMs with DRA-backed devices
- OVS-DPDK: Open vSwitch Data Plane Development Kit — a user-space switch, bypassing the Linux kernel network stack, and used as the concrete test implementation for this STP
- ODC: OvsDpdkConfig — the CRD used to configure the OVS-DPDK DRA driver ([definition](https://github.com/k8snetworkplumbingwg/dra-driver-ovsdpdk/#ovsdpdkconfig-cluster-scoped))
- ODRP: OvsDpdkResourcePolicy — the CRD that maps OVS-DPDK bridges to DRA-allocatable resources ([definition](https://github.com/k8snetworkplumbingwg/dra-driver-ovsdpdk/#ovsdpdkresourcepolicy-namespaced))

### **Feature Overview**

OVS-DPDK (Open vSwitch with the DPDK data plane) delivers high-throughput, low-latency virtual networking by bypassing the Linux kernel and processing packets in userspace. This STP covers the Tech Preview introduction of OVS-DPDK as a network binding plugin for OpenShift Virtualization VMs in OCP 5.1, using Kubernetes DRA as the device provisioning mechanism. VM owners reference OVS-DPDK vhost-user ports through DRA resource claims (ResourceClaim or ResourceClaimTemplate) and attach them to VMs via the OVS-DPDK vhost-user binding plugin; KubeVirt orchestrates the interaction between the DRA driver and the binding plugin.

> **Note on TP testing scope:** Although this feature ships as Tech Preview in OCP 5.1, QE is providing full test coverage as an exceptional commitment to the target customer. This is not the standard TP testing approach.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:*
    - VMs can attach OVS-DPDK vhost-user interfaces using ResourceClaim or ResourceClaimTemplate with the OVS-DPDK vhost-user binding plugin.
    - Live migration is supported for VMs with ResourceClaimTemplate-backed OVS-DPDK devices.
    - The `NetworkDevicesWithDRA` feature gate gates all DRA network API surface; it is enabled through the HyperConverged CR.
    - The OVS-DPDK DRA driver and vhost-user binding plugin images are shipped via Konflux.
    - Invalid or unsupported configurations (missing binding plugin, mixed sources, empty claim references) are rejected at admission time.

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* OVS-DPDK delivers wire-speed virtual networking for performance-intensive workloads by bypassing the kernel network stack. Using DRA as the provisioning model gives VM owners a Kubernetes-native, claim-based approach to attaching high-performance OVS-DPDK vhost-user interfaces without custom OpenShift Virtualization configuration.
  - *List the customer use cases identified:*
    - As a VM owner, I would like to run a DPDK application in my VM by using OVS-DPDK network binding, bypassing both host and guest kernel.
    - As a VM owner, I would like to be able to live-migrate an OVS-DPDK based VM.
    - As a cluster admin, I would like to enable DPDK network acceleration.

- [x] **Testability**
  - *Note any requirements that are unclear or untestable:* Core flows are testable. The OVS-DPDK DRA driver setup is applied at cluster deployment time (day-0); see CNV-96151 for the CI lane implementation tracking issue.

- [x] **Acceptance Criteria**
  - *List the acceptance criteria:*
    1. A VM with a ResourceClaimTemplate-backed DRA network device and a network binding plugin starts successfully and has connectivity through the assigned device.
    2. A VM with a direct ResourceClaim-backed DRA network device and a network binding plugin starts successfully and has connectivity through the assigned device.
    3. A VM with a ResourceClaimTemplate-backed DRA network device can be live-migrated; an active connection established before migration is maintained or re-established after migration completes, and the VM is reachable on the destination node.
    4. When the NetworkDevicesWithDRA feature gate is disabled, a VM spec referencing a DRA network source is rejected at admission time with a clear error.
    5. A VM spec that mixes DRA and Multus-based network sources is rejected at admission time with a clear error.
    6. The live migration request for a VM with a direct ResourceClaim-backed DRA network device is rejected at admission or scheduling time with a clear error, confirmed by inspecting the rejection event. Execution-time behavior is not validated for TP due to test-safety constraints.
    7. A VM interface that references a DRA network source but has no network binding plugin configured is rejected at admission time with a clear error.
    8. VMs with DRA-backed OVS-DPDK network devices on a bridge configured with a non-default MTU can exchange traffic using the full jumbo-frame payload size, with no fragmentation.
  - *Note any gaps or missing criteria:* None for TP scope.

- [x] **Non-Functional Requirements (NFRs)**
  - *List applicable NFRs and their targets:*
    - Documentation: User guide for DRA network device setup and VM configuration required for TP release.
    - Security: No new RBAC or authentication changes introduced; existing Kubernetes DRA access controls and webhook validations apply.
    - Scalability: No new scale requirements for TP phase; the feature follows the existing DRA scalability model from Kubernetes.
    - Monitoring/Observability: No new metrics or alerts required for TP phase.
    - UI: Feature is API-driven; no new UI components required. UI team confirmed no testing needed for TP.
    - Performance: No performance requirements defined for TP phase; deferred to GA.
  - *Note any NFRs not covered and why:*
    - Portability (cloud): OVS-DPDK requires bare metal; cloud platform testing is not applicable.

#### **2. Known Limitations**

- **Hot-plug/hot-unplug:** Adding or removing DRA-backed network interfaces on a running VM is not supported.
  - *Sign-off:* [Name/Date]

- **ResourceClaim live migration:** Live migration is only supported for VMs whose DRA network devices were allocated via ResourceClaimTemplate. VMs using direct ResourceClaim-backed devices cannot be live-migrated.
  - *Sign-off:* [Name/Date]

- **Mixed network sources:** Mixing DRA and Multus-based network sources on the same VM is not supported.
  - *Sign-off:* [Name/Date]

- **SR-IOV:** SR-IOV is not a supported DRA network device type in this release; support may be revisited once industry approaches for SR-IOV + DRA integration converge.
  - *Sign-off:* [Name/Date]

- **Multiple devices per claim:** Allocating multiple network devices via a single ResourceClaim or ResourceClaimTemplate (count > 1) has not been validated and is not a supported configuration.
  - *Sign-off:* [Name/Date]

- **MAC address configuration:** MAC address specification for DRA-backed network interfaces is outside the scope of this feature; MAC assignment is the responsibility of the DRA driver or binding plugin.
  - *Sign-off:* [Name/Date]

- **DPDK applications inside VMs:** Running DPDK workloads inside a VM (vfio passthrough of the vhost-user device) requires vIOMMU and hugepages, hence testing is limited to QE bare-metal clusters.
  - *Sign-off:* [Name/Date]

#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:*
    - DRA replaces the traditional device-plugin approach: users describe the network device they need using Kubernetes resource claim objects (ResourceClaim or ResourceClaimTemplate), and the Kubernetes scheduler allocates devices with full awareness of device constraints and placement requirements.
    - Once the DRA driver provisions the device, a network binding plugin configures the VM's network interface. KubeVirt orchestrates between the DRA driver and the binding plugin.
    - The TP phase validates OVS-DPDK integration only; support for additional DRA network drivers is deferred to GA.
    - A feature gate gates all new API surface; disabling the gate rejects any DRA network source at admission time.
    - Rollback from an enabled feature gate requires removing all VMs using DRA network devices before disabling the gate.

- [x] **Technology Challenges**
  - *List identified challenges:*
    - Testing requires a functional external OVS-DPDK DRA driver. Setting up OVS-DPDK as a DRA driver involves significant infrastructure configuration (hugepages, OVS configuration, DPDK-capable NICs or software equivalent).
    - The OVS-DPDK DRA driver setup is applied at cluster deployment time (day-0). This approach was selected to ensure environment consistency across test runs; see CNV-96151 for the CI lane implementation tracking issue.
    - The feature requires Kubernetes 1.34+ for network binding plugin support. OCP 5.1 must meet this version requirement.
  - *Impact on testing approach:* Tests require a dedicated environment with the OVS-DPDK DRA driver deployed and validated before test execution. A standard shared CI lane is not suitable without driver setup; a dedicated CI lane is needed.

- [x] **API Extensions**
  - *List new or modified APIs:* A new DRA network source type is added to the VM network configuration API. Users reference their DRA resource claims from the VM spec using the new source type. A feature gate controls availability of this new source type.
  - *Testing impact:* Test scenarios cover both valid DRA network configurations and invalid ones (rejected by admission webhooks). Existing tests for Multus-based networks are unaffected.

- [x] **Test Environment Needs**
  - *See environment requirements in Section II.3 and testing tools in Section II.3.1*

- [x] **Topology Considerations**
  - *Describe topology requirements:* All tests run on a bare-metal cluster (3-master/3-worker) with DPDK-capable physical NICs and IOMMU enabled on worker nodes. OVS-DPDK requires real hardware and cannot run on a virtualized cluster. Multi-node topology is required for live migration scenarios.
  - *Impact on test design:* All scenarios run on the bare-metal cluster. Live migration scenarios require at least two worker nodes, which the 3-worker topology satisfies.

### **II. Software Test Plan (STP)**

#### **1. Scope of Testing**

**Testing Goals**

- **[P0]** Verify that a VM can be created with an OVS-DPDK DRA network device allocated via ResourceClaimTemplate with a network binding plugin and establish connectivity through the assigned device.
- **[P0]** Verify that when the NetworkDevicesWithDRA feature gate is disabled, any VM spec referencing a DRA network source is rejected at admission time with a clear error — confirming the feature gate correctly gates the capability.
- **[P0]** Verify that a VM with a ResourceClaimTemplate-backed OVS-DPDK DRA network device can be live-migrated between nodes with an active connection maintained or re-established after migration completes.
- **[P0]** Verify that the live migration request for a VM with a direct ResourceClaim-backed DRA network device is rejected at admission or scheduling time with a clear error message, confirmed by inspecting the rejection event.
- **[P1]** Verify that a VM can be created with an OVS-DPDK DRA network device allocated via direct ResourceClaim with a network binding plugin and establish connectivity through the assigned device.
- **[P1]** Verify that a VM with two DRA-backed OVS-DPDK vhost-user interfaces (each from a separate ResourceClaimTemplate) can be created and traffic flows independently on each interface.
- **[P1]** Verify that when the OVS-DPDK DRA driver is restarted, VMs with existing vhost-user ports remain functional and new VMs with DRA network devices can be created.
- **[P1]** Verify that when a VM with a DRA-backed vhost-user port is deleted, all associated host-level resources (OVS ports, vhost-user sockets) are fully cleaned up.
- **[P1]** Verify that VMs with OVS-DPDK DRA network devices on a bridge with non-default MTU can send traffic with jumbo frames between each other, with no fragmentation.
- **[P1]** Verify that VM specs mixing DRA and Multus-based network sources on the same VM are rejected at admission time with a clear error.
- **[P1]** Verify that a VM interface configured with a DRA network source but without a network binding plugin is rejected at admission time with a clear error.
- **[P1]** Verify that VM specs with invalid DRA network configuration (empty or missing claim name or request name, or two networks referencing the same claimName+requestName combination) are rejected at admission time.
- **[P2]** Verify that VMs with OVS-DPDK DRA network devices can be live-migrated with connectivity preserved after the OVS-DPDK DRA driver has been restarted on both source and destination nodes.
- **[P2]** Verify that two VMs on different nodes, each allocated an OVS-DPDK DRA network device from the same DeviceClass via separate ResourceClaimTemplates, can start and communicate over their respective DRA-backed network interfaces.
- **[P2]** Verify that a VM with a ResourceClaimTemplate-backed DRA network device remains functional after an OCP minor-version upgrade; connectivity is re-established after the upgrade completes.

**Out of Scope (Testing Scope Exclusions)**

- **Hot-plug/hot-unplug of DRA-backed network interfaces**
  - *Rationale:* Not supported in this release. No test coverage planned for TP.
  - *PM/Lead Agreement:* [Name/Date]

- **SR-IOV as a DRA network device type**
  - *Rationale:* SR-IOV DRA integration is out of scope for this release due to competing industry approaches. OVS-DPDK is the only concrete DRA driver under test.
  - *PM/Lead Agreement:* [Name/Date]

- **Performance benchmarking of OVS-DPDK throughput and latency**
  - *Rationale:* No performance requirements are defined for the TP phase; performance validation is deferred to GA.
  - *PM/Lead Agreement:* [Name/Date]

- **ResourceClaimTemplate with count > 1 (multiple devices per claim request)**
  - *Rationale:* This scenario has not been validated and is not a supported configuration for this release.
  - *PM/Lead Agreement:* [Name/Date]

- **Device Plugin NUMA-aware scheduling for OVS-DPDK resources**
  - *Rationale:* Exposing OVS-DPDK resources via the legacy Device Plugin for NUMA-aware pod/VM scheduling is a separate mechanism from DRA. It requires bare metal with dual-NUMA nodes. This STP covers DRA-based device consumption only.
  - *PM/Lead Agreement:* [Name/Date]

**Test Limitations**

- **DPDK applications inside VMs:** Testing DPDK workloads inside VMs (vfio passthrough of the vhost-user device) requires vIOMMU and hugepages, hence testing is limited to QE bare-metal clusters. This use case is documented in Known Limitations (Section I.2).
  - *Sign-off:* [Name/Date]

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** — Validates that the feature works according to specified requirements and user stories
  - *Details:* Validate VM creation with OVS-DPDK DRA network devices (via RC and RCT), connectivity establishment, live migration with RCT-backed devices, feature gate enforcement, and admission-time rejection of invalid configurations.

- [x] **Automation Testing** — Confirms test automation plan is in place for CI and regression coverage (all tests are expected to be automated)
  - *Details:* All new test scenarios will be automated. A dedicated CI lane is required due to OVS-DPDK hardware and driver setup requirements; the standard shared lane cannot be used without driver setup (see Section II.3.1).

- [x] **Regression Testing** — Verifies that new changes do not break existing functionality
  - *Details:* Existing secondary network and live migration test suites run as regression to confirm no impact from the new DRA network source type. Enabling the feature gate must not affect existing VMs using Multus-based networks.

- [ ] **Self-Validation Testing** — Should any of the new tests be included in the self-validation test package?
  - *Details:* Not applicable. DRA network device tests require a dedicated OVS-DPDK driver setup and special hardware; they are not suitable for the self-validation package.

**Non-Functional**

- [ ] **Performance Testing** — Validates feature performance meets requirements (latency, throughput, resource usage)
  - *Details:* Not applicable for TP phase; no performance requirements defined. Deferred to GA.

- [ ] **Scale Testing** — Validates feature behavior under increased load and at production-like scale
  - *Details:* Not applicable for TP phase; no scale requirements defined. The feature follows the existing DRA scalability model.

- [ ] **Security Testing** — Verifies security requirements, RBAC, authentication, authorization, and vulnerability scanning
  - *Details:* Not applicable; no new RBAC or authentication changes introduced. Webhook admission validation is exercised as part of Functional Testing (invalid config rejection scenarios).

- [ ] **Usability Testing** — Validates user experience and accessibility requirements
  - *Details:* Not applicable. Feature is API-driven; no new UI components or CLI commands introduced. UI team confirmed no testing required for TP phase.

- [ ] **Monitoring** — Does the feature require metrics and/or alerts?
  - *Details:* Not applicable for TP phase; no new metrics or alerts introduced.

**Integration & Compatibility**

- [x] **Compatibility Testing** — Ensures feature works across supported platforms, versions, and configurations
  - *Details:* Validate that the feature gate can be toggled between enabled and disabled. Existing Multus-based VM tests run as regression to confirm no impact from enabling the feature gate. Confirm the Kubernetes 1.34+ version requirement is met in OCP 5.1.

- [x] **Upgrade Testing** — Validates upgrade paths from previous versions, data migration, and configuration preservation
  - *Details:* Validate that a VM with a DRA-backed network device (RCT-backed) remains functional after an OCP minor-version upgrade with connectivity re-established after the upgrade (see P2 testing goal). Validate the rollback path: VMs using DRA network devices can be removed and the feature gate disabled cleanly.

- [x] **Dependencies** — Blocked by deliverables from other components/products
  - *Details:* Testing depends on a stable external OVS-DPDK DRA driver being available for QE use. Network binding plugin must be registered and functional before test execution begins. Track driver and binding plugin readiness against the CNV-88618 epic.

- [x] **Cross Integrations** — Does the feature affect other features or require testing by other teams?
  - *Details:* The network binding plugin team must validate correct integration with DRA device attributes. The live migration flow is affected; any regressions in VM scheduling or migration with DRA devices must be caught during testing.

**Infrastructure**

- [ ] **Cloud Testing** — Does the feature require multi-cloud platform testing?
  - *Details:* Not applicable. OVS-DPDK requires bare metal; cloud platforms are not supported for this feature.

#### **3. Test Environment**

**Environment — Bare metal cluster (all tests)**

- **Cluster Topology:** 3-master/3-worker bare metal cluster; multi-node worker topology required for live migration scenarios
- **OCP & OpenShift Virtualization Version(s):** OCP 5.1 with OpenShift Virtualization 5.1 (Kubernetes server version ≥ 1.34)
- **CPU:** Hardware virtualization enabled (VT-x / AMD-V); IOMMU enabled (VT-d / AMD-Vi) on all worker nodes
- **Compute Resources:** Minimum per worker node: 32 physical CPUs, 64 GB RAM; hugepages configured for OVS-DPDK
- **Special Hardware:** DPDK-capable physical NICs on worker nodes; dual-NUMA node topology available on workers
- **Storage:** ocs-storagecluster-ceph-rbd-virtualization (ReadWriteMany access mode, Block volume mode — required for live-migration scenarios)
- **Network:** OVN-Kubernetes, IPv4; OVS-DPDK configured as a DRA network device on worker nodes
- **Required Operators:** N/A
- **Platform:** Bare metal
- **Special Configurations:** OVS-DPDK DRA driver deployed and configured on worker nodes at cluster provisioning time (day-0; see CNV-96151); `NetworkDevicesWithDRA` feature gate enabled; network binding plugin registered in the HyperConverged CR.
- **Not suitable for:** Cloud platforms (OVS-DPDK requires bare metal hardware).

#### **3.1. Testing Tools & Frameworks**

- **Test Framework:** Standard

- **CI/CD:** Dedicated CI lane required — OVS-DPDK DRA driver setup cannot be assumed in standard shared lanes

- **Other Tools:** OVS-DPDK DRA driver (external dependency)

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [ ] Requirements and design documents are **approved and merged**
- [ ] OCP 5.1 Kubernetes server version confirmed ≥ 1.34 (verified via `kubectl version`)
- [ ] Test environment is **set up and configured** with OVS-DPDK DRA driver deployed and functional (see Section II.3)
- [ ] `NetworkDevicesWithDRA` feature gate is available and can be toggled in the test cluster
- [ ] Network binding plugin is registered for the OVS-DPDK device type

#### **5. Risks**

**Timeline/Schedule**

- **Risk:** The OVS-DPDK DRA driver and test environment setup are complex and require cross-team coordination. The setup approach (day-0) has been decided; CI lane implementation is tracked under [CNV-96151](https://redhat.atlassian.net/browse/CNV-96151).
  - **Mitigation:** CI lane implementation tracked under CNV-96151.
  - *Estimated impact on schedule:* N/A — decision resolved.
  - *Sign-off:* [Name/Date]

**Test Coverage**

- **Risk:** Testing is scoped to OVS-DPDK as the only concrete DRA network driver for this release. Other DRA network drivers, if introduced in future releases, will require separate test coverage.
  - **Mitigation:** Design test assertions against user-observable outcomes (connectivity, migration success, admission rejection) rather than OVS-DPDK-specific internals, so tests remain applicable to future drivers.
  - *Areas with reduced coverage:* Any DRA network driver type other than OVS-DPDK.
  - *Sign-off:* [Name/Date]

**Test Environment**

- **Risk:** The bare-metal cluster requires DPDK-capable physical NICs, IOMMU, and hugepages configuration. Environment provisioning is complex and depends on infrastructure team support and hardware availability.
  - **Mitigation:** Environment setup is treated as a day-0 activity tracked under CNV-96151. Hardware requirements are documented explicitly so provisioning can start early.
  - *Missing resources or infrastructure:* Bare metal cluster with DPDK-capable NICs required; tracked under CNV-96151.
  - *Sign-off:* [Name/Date]

**Untestable Aspects**

- **Risk:** Validating that live migration of a VM with a direct ResourceClaim-backed DRA device (an unsupported scenario) fails safely is difficult without risking VM corruption during testing.
  - **Mitigation:** Limit testing to verifying observable rejection or clear failure at the admission or scheduling layer. Avoid driving the migration path for this unsupported configuration into states that could harm the workload.
  - *Reason untestable and mitigation approach:* The migration path for RC-backed devices may not fail cleanly at all layers; testing deeply may result in workload loss. Testing at the admission/scheduling boundary is sufficient to confirm the non-goal is enforced.
  - *Sign-off:* [Name/Date]

**Resource Constraints**

- **Risk:** Setting up and maintaining the OVS-DPDK DRA driver configuration on the bare-metal cluster requires cross-team expertise spanning OVS-DPDK and KubeVirt DRA. Insufficient expertise within QE may slow environment setup and test development.
  - **Mitigation:** Coordinate with the development team for environment setup assistance during the initial test development phase. Document the setup procedure (ODC/ODRP configuration, driver deployment) to reduce ongoing dependency.
  - *Current capacity gaps:* OVS-DPDK DRA driver operational expertise within QE.
  - *Sign-off:* [Name/Date]

**Dependencies**

- **Risk:** Test execution depends on a stable and functional external OVS-DPDK DRA driver being available for QE use. If the driver is not ready or stable, all DRA network device test scenarios are blocked.
  - **Mitigation:** Track driver readiness against the CNV-88618 epic; do not begin test execution until the driver is confirmed stable for QE use. Coordinate with the DRA driver team for early access to a test build.
  - *Dependent teams or components:* OVS-DPDK DRA driver team; network binding plugin team; KubeVirt DRA core team.
  - *Sign-off:* [Name/Date]

---

### **III. Test Scenarios & Traceability**

| Requirement ID   | Requirement Summary | Test Scenario(s) | Tier | Priority |
|:-----------------|:--------------------|:-----------------|:-----|:---------|
| CNV-88618 (epic) | As a VM owner, I want to attach DRA-managed network devices to my VM using a ResourceClaimTemplate so I can consume them the same way I would in a container workload | Verify that a VM created with a ResourceClaimTemplate-backed OVS-DPDK DRA network device and a network binding plugin starts successfully and has network connectivity through the assigned device; verify that two such VMs can ping and exchange traffic with each other | Tier 1 | P0 |
|                  | As a VM owner, I want to live-migrate my VM with a DRA-backed network device so I can perform maintenance without VM downtime | Verify that a VM with a ResourceClaimTemplate-backed OVS-DPDK DRA network device can be live-migrated between nodes; an active connection established before migration is maintained or re-established within normal migration bounds after migration completes, and the VM is reachable on the destination node | Tier 2 | P0 |
|                  | As a VM owner, I want to attach DRA-managed network devices to my VM using a direct ResourceClaim so I can consume them the same way I would in a container workload | Verify that a VM created with a direct ResourceClaim-backed OVS-DPDK DRA network device and a network binding plugin starts successfully and has network connectivity through the assigned device | Tier 1 | P1 |
|                  | As a VM owner, I want my DRA-backed network device to remain usable after a cluster upgrade | Verify that a VM with a ResourceClaimTemplate-backed OVS-DPDK DRA network device remains running after an OCP minor-version upgrade and network connectivity can be re-established after the upgrade completes | Tier 2 | P2 |

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - Development: Nir Dothan (@nirdothan), Ananya Banerjee (@frenzyfriday), Bruno Gomes (@brunogomes011), Maxime Coquelin (@mcoqueli), Adrian Moreno (@amorenoz)
  - QE: Asia Khromov (@azhivovk), Yossi Segev (@yossisegev)
* **Approvers:**
  - QE Lead: Ruth Netser (@rnetser)
  - Dev Lead: Orel Misan (@orelmisan)
  - Product Manager: Ronen Sde-Or (@ronensdeor)
