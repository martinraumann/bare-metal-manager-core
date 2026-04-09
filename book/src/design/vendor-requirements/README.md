# Vendor Requirements — Design Specification

**Status:** DRAFT  
**Authors:** M. Raumann, D. Porokh  
**Last Updated:** 2026-04-09  

---

## 1. Introduction

This document proposes a **struct-based vendor requirements model** that replaces the growing body
of vendor- and model-specific hardware workarounds hardcoded across `bare-metal-manager-core`.

The core idea is a separation of concerns: vendor modules *declare* what a platform requires, and
carbide's execution engine *acts* on those requirements. Vendor knowledge is expressed as data
rather than imperative code, which isolates each contributor's changes from every other vendor's
execution path.

### 1.1 Scope

This document covers:

- The structural problem with the current approach and why it becomes intractable in open source (§2).
- An inventory of the 8+ existing hardcoded workarounds across four files (§3).
- Three approaches considered and the rationale for the recommended hybrid (§4).
- The `VendorRequirements` struct design, vendor file layout, and centralized execution engine (§5).
- Relationship to the Convergence Engine RFC (PR #846) and how the two proposals interact (§6).
- Testing strategy, migration plan, and open questions (§7–§9).

The companion guide [`book/src/development/new_hardware_support.md`](../development/new_hardware_support.md)
documents the current process for adding hardware support. This RFC targets specifically the
**"Changes in NICo"** section of that guide: the `VendorRequirements` model replaces the
pattern of adding `bmc_vendor().is_*()` branches in `handler.rs` with isolated struct
declarations. When this RFC is implemented, that section of the guide should be updated
to reflect the new contribution path.

### 1.2 Relationship to the Convergence Engine RFC

PR #846 (I. Anisimov) proposes replacing the entire FSM-based state handler system with a
declarative convergence engine. This proposal is narrower: it targets the pre-ingestion and
behavioral-flag layer specifically, without touching the FSM, and is implementable now.

The two proposals are **complementary, not competing**. If the convergence engine ships, the vendor
functions defined here become the first profile modules. See §6 for a detailed mapping.

---

## 2. Problem Statement

### 2.1 Current Architecture

`bare-metal-manager-core` manages the lifecycle of bare-metal hosts through a state machine. Hardware
vendor support is implemented by adding conditional branches in shared handler files:

```
preingestion_manager/mod.rs   — pre-ingestion hardware setup
handlers/instance.rs          — instance allocation and reboot
handlers/machine.rs           — force delete, UEFI clear
redfish.rs                    — restart protocol, boot order
site_explorer/redfish.rs      — BMC password change, OOB MAC
```

Each vendor-specific workaround is a hardcoded `if vendor == X` or `if model.contains("Y")`
branch in one of these shared files. The BMC vendor routing layer compounds this:

```
Incoming BMC  →  TLS issuer  →  BMCVendor enum  →  HwType enum  →  execution paths
```

`HwType::Ami` is a catch-all: any AMI-based machine that isn't a Viking DGX lands there.
Changing any behavior in that path affects every AMI machine in the fleet.

### 2.2 Pain Points

1. **Blast radius is structural.** A contributor changing the generic AMI path to fix their hardware
   can break every other AMI machine — not through carelessness but because the architecture does
   not isolate their change.

2. **Intractable in open source.** As soon as contributor #2 arrives with hardware the core team
   does not own, there is no way to regression-test whether their change breaks contributor #1's
   hardware, or vice versa. The problem is O(n²) in hardware diversity and physically impossible
   to fully test. (See Slack thread, 2026-04-09: *"Two trains full of burning dumpsters rushing
   towards each other."* — M. Heyeck)

3. **BMCVendor misclassification.** The TLS issuer `"American Megatrends International LLC (AMI)"`
   maps to `BMCVendor::Nvidia`. A Gigabyte AMI machine is therefore classified as NVIDIA, which
   can trigger Viking-specific password logic (hardcoded account ID `"2"`) and Viking-specific IPMI
   console paths that have nothing to do with Gigabyte hardware.

4. **No path for external contributors.** There is no documented, safe way for an external vendor
   to add support for their hardware without touching shared routing and execution logic. The
   current process is: try to ingest, see where it fails, add an if-statement. That is not
   something we can hand to an external contributor and call a contribution guide.

5. **Compilation time.** A naive fix — a vendor trait with async methods — would repeat the
   known `libredfish` compilation penalty (~3 seconds per new platform from `async_trait` macro
   expansion). See §4.3.

### 2.3 Forcing Function: The Gigabyte Case

Gigabyte (external customer) has an issue with their AMI BMC code and wants to remove the
`EndlessBoot` requirement on the generic AMI path. That path (`HwType::Ami`) currently sets
`EndlessBoot = "Enabled"` for all AMI machines that are not Viking DGX. Removing it globally
could break LenovoAMI and any future AMI hardware the team enables that needs it.

This is not a Gigabyte-specific problem. It is the first visible instance of a structural issue
that will recur with every new external hardware contributor.

### 2.4 Desired Properties

| Property | Description |
|---|---|
| **Isolated** | A contributor's change cannot affect another vendor's execution path. |
| **Declarative** | Vendor knowledge is expressed as data, not imperative code. |
| **Testable** | Vendor requirements and execution logic are independently testable. |
| **Config-migratable** | The same data shape works in Rust and TOML — no separate mapping layer. |
| **Low compile overhead** | No growing async trait; no macro expansion per new vendor. |
| **Incrementally adoptable** | Can be implemented in phases without touching the FSM. |

---

## 3. Inventory of Existing Hardcoded Workarounds

A full audit found at least 8 vendor-specific workarounds across four files.

### Category A — Pre-Ingestion Hardware Configuration

Direct hardware attribute or state setup before a machine enters production.

| # | Location | Behavior | Trigger |
|---|---|---|---|
| A1 | `preingestion_manager/mod.rs` | Dell R760: disable PSU Hot Spare via Redfish attribute | Initial state, after time sync |
| A2 | `preingestion_manager/mod.rs` | Lenovo: AC power cycle during firmware drain | `ResetForNewFirmware`, `PowerDrainState::Off` |
| A3 | `preingestion_manager/mod.rs` | Lenovo + NVIDIA: BMC reset after BMC firmware upgrade | Post-upgrade, BMC component only |

### Category B — Runtime Behavioral Flags

Differences in how standard operations execute per vendor.

| # | Location | Behavior | Trigger |
|---|---|---|---|
| B1 | `handlers/instance.rs` | Lenovo: `boot_first` instead of `boot_once` (avoids PCIe reset) | Instance reboot |
| B2 | `redfish.rs` | Lenovo: `boot_once(PXE)` before power action to clean boot order | Machine restart |
| B3 | `handlers/machine.rs` | Lenovo: restart host after UEFI password clear | Force delete |
| B4 | `handlers/instance.rs` | Lenovo: skip BMC lockdown (not supported) | Instance allocation |

### Category C — Conditional Logic Requiring Runtime Data

| # | Location | Behavior | Why It's Hard |
|---|---|---|---|
| C1 | `redfish.rs` | NVIDIA Viking DGX: use IPMI instead of Redfish for restarts | Must fetch `service_root`, `system`, and `manager` from Redfish to detect Viking, then switch protocols entirely |

### Category D — Discovery-Layer Workarounds (Out of Scope)

| # | Location | Behavior |
|---|---|---|
| D1 | `site_explorer/redfish.rs` | 5 different vendor implementations for BMC password change API |
| D2 | `site_explorer/redfish.rs` | Bluefield: OOB MAC retrieved via boot options parsing (no Redfish API) |

D1 and D2 are discovery-layer concerns and belong in the Redfish client layer, not the
pre-ingestion system. They are tracked separately.

### The Heat Map

| Function | Lenovo | NVIDIA Viking | Dell | Generic AMI |
|---|---|---|---|---|
| Firmware drain | AC power cycle | — | — | — |
| BMC reset post-upgrade | Required | Required | — | — |
| Boot sequence | `boot_first` + PXE cleanup | — | — | `EndlessBoot` attr |
| Restart protocol | — | IPMI (not Redfish) | — | — |
| BMC lockdown | Not supported | — | — | — |
| UEFI password clear | Restart required | — | — | — |
| Pre-ingestion attributes | — | — | PSU Hot Spare | — |
| Password change API | Username-based | Account ID `"2"` | Username-based | → misrouted as NVIDIA |

---

## 4. Approaches Considered

Three approaches were evaluated. The recommended solution is the hybrid struct-based model (§4.3).

### 4.1 Approach 1 — Pure Config

Extend the existing `host_models` TOML section with `preingestion_actions` entries and capability
flags.

**Pros:** Operators can add Redfish attribute actions without a code change. Config is auditable
by non-Rust engineers.

**Cons:** Action parameters are stringly-typed — typos cause runtime failures, not compile errors.
Complex conditional logic (Category C) requires a named-handler escape hatch.

### 4.2 Approach 2 — Vendor Trait Files

Organize vendor-specific logic into dedicated Rust files behind a common trait.

**Pros:** Type-safe, compile-time verified. Clear ownership per vendor file.

**Cons:** A growing `async` trait repeats the `libredfish` compilation time problem exactly
(~3 seconds per new platform). Every new vendor must stub all trait methods. Adding a new
requirement field forces touching all existing vendor implementations.

### 4.3 Approach 3 — Hybrid Struct-Based (Recommended)

A plain Rust struct with `#[derive(Default, Deserialize)]`. Vendors declare what they need;
carbide acts on those requirements. No trait, no `async_trait`, no compilation overhead.

**Key differences from Approach 2:**

- No trait → no `async_trait` macro expansion → no compilation overhead per new vendor
- New fields use `..Default::default()` → no existing vendor code needs updating
- Same struct deserializes from TOML → config and code share identical data shape
- Vendor files are simple functions returning structs, not trait implementations

**Prior art in the codebase:** The `bios_profiles` / `machine_setup` system already demonstrates
this pattern — a struct deserialized from TOML, keyed by vendor and model, executed by carbide's
state controller. `VendorRequirements` is the generalization of that existing pattern.

---

## 5. The Struct-Based Requirements Model

### 5.1 The Requirements Struct

```rust
/// Describes what a vendor/model requires.
/// Carbide owns all execution logic; this struct carries only requirements.
/// Adding a new field never requires touching existing vendor files.
#[derive(Default, Clone, Deserialize)]
pub struct VendorRequirements {
    // Firmware upgrade behavior
    #[serde(default)]
    pub needs_bmc_reset_after_bmc_upgrade:  bool,
    #[serde(default)]
    pub needs_ac_power_cycle_during_drain:  bool,

    // Boot sequence
    #[serde(default)]
    pub boot_order_behavior:                BootOrderBehavior,

    // Security / lockdown
    #[serde(default = "default_true")]
    pub supports_bmc_lockdown:              bool,
    #[serde(default)]
    pub needs_restart_after_uefi_clear:     bool,

    // Restart protocol
    #[serde(default)]
    pub restart_protocol:                   RestartProtocol,

    // Declarative Redfish attributes (Category A)
    #[serde(default)]
    pub preingestion_attributes:            Vec<RedfishAttributeSet>,
}

#[derive(Default, Clone, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum BootOrderBehavior {
    #[default] Standard,
    LenovoCleanup,  // boot_once(PXE) then power
    BootFirst,
}

#[derive(Default, Clone, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum RestartProtocol {
    #[default] StandardRedfish,
    IpmiRequired,   // Viking DGX case
}

#[derive(Clone, Deserialize)]
pub struct RedfishAttributeSet {
    pub path:       String,
    pub attributes: HashMap<String, String>,
    #[serde(default)]
    pub required:   bool,
}
```

### 5.2 Vendor Files as Simple Functions

Vendor files are not trait implementations — they are functions that return a requirements struct.
No async, no macros, no compilation overhead.

```rust
// vendors/lenovo.rs
pub fn requirements(_model: &str) -> VendorRequirements {
    VendorRequirements {
        needs_bmc_reset_after_bmc_upgrade: true,
        needs_ac_power_cycle_during_drain: true,
        supports_bmc_lockdown:             false,
        boot_order_behavior:               BootOrderBehavior::LenovoCleanup,
        needs_restart_after_uefi_clear:    true,
        ..Default::default()
    }
}

// vendors/dell.rs
pub fn requirements(model: &str) -> VendorRequirements {
    let mut reqs = VendorRequirements::default();
    if model.to_lowercase().contains("r760") {
        reqs.preingestion_attributes.push(RedfishAttributeSet {
            path: "/redfish/v1/Managers/System.Embedded.1/Attributes".into(),
            attributes: [("ServerPwr.1.PSRapidOn".into(), "Disabled".into())].into(),
            required: false,
        });
    }
    reqs
}

// vendors/nvidia.rs
pub fn requirements(model: &str) -> VendorRequirements {
    let mut reqs = VendorRequirements {
        needs_bmc_reset_after_bmc_upgrade: true,
        ..Default::default()
    };
    if model.to_lowercase().contains("dgx") {
        reqs.restart_protocol = RestartProtocol::IpmiRequired;
    }
    reqs
}

// vendors/mod.rs — one match, no trait, no dynamic dispatch
pub fn vendor_requirements(vendor: BMCVendor, model: &str) -> VendorRequirements {
    match vendor {
        BMCVendor::Dell   => dell::requirements(model),
        BMCVendor::Lenovo => lenovo::requirements(model),
        BMCVendor::Nvidia => nvidia::requirements(model),
        _                 => VendorRequirements::default(),
    }
}
```

### 5.3 Config Override

Because `VendorRequirements` derives `Deserialize`, the same struct maps directly from TOML.
The execution engine does not know or care whether requirements came from Rust or config.

```rust
pub fn get_requirements(
    vendor:      BMCVendor,
    model:       &str,
    host_config: Option<&HostModelConfig>,
) -> VendorRequirements {
    if let Some(cfg) = host_config {
        if let Some(reqs) = &cfg.requirements {
            return reqs.clone();  // config takes precedence
        }
    }
    vendor_requirements(vendor, model)
}
```

```toml
# carbide.toml — same struct, deserialized from config instead of Rust
[host_models.lenovo_sr670v2.requirements]
needs_bmc_reset_after_bmc_upgrade = true
needs_ac_power_cycle_during_drain = true
supports_bmc_lockdown             = false
boot_order_behavior               = "lenovo_cleanup"
needs_restart_after_uefi_clear    = true

[[host_models.dell_r760.requirements.preingestion_attributes]]
path       = "/redfish/v1/Managers/System.Embedded.1/Attributes"
required   = false
attributes = { "ServerPwr.1.PSRapidOn" = "Disabled" }
```

### 5.4 Centralized Execution Engine

Carbide owns all action logic. The execution engine reads requirements and acts — it never
inspects vendor identity directly.

```rust
// preingestion_manager/mod.rs
let reqs = get_requirements(endpoint.vendor, &endpoint.model, host_config.as_ref());

for attr_set in &reqs.preingestion_attributes {
    apply_redfish_attributes(attr_set, &redfish_client).await;
}
if reqs.needs_bmc_reset_after_bmc_upgrade && upgrade_type.is_bmc() {
    redfish_client.bmc_reset().await?;
}
match reqs.restart_protocol {
    RestartProtocol::StandardRedfish => redfish_client.power(action).await?,
    RestartProtocol::IpmiRequired    => ipmi_tool.restart(...).await?,
}
```

---

## 6. Relationship to the Convergence Engine RFC

PR #846 proposes replacing the FSM-based state handlers with a declarative convergence engine
using hardware profiles, an `op!` macro DSL, and a three-predicate scheduler.

### 6.1 These Proposals Are Complementary

| | This Proposal | Convergence Engine (PR #846) |
|---|---|---|
| Scope | Pre-ingestion workarounds + behavioral flags | Entire lifecycle FSM, all object types |
| Mechanism | Plain struct, vendor functions | `op!` macro DSL, profile inheritance, scheduler |
| Timeline | Implementable now | RFC stage; major migration |
| Gigabyte fix | Yes, directly | No near-term relief |
| Risk | Low, phased | High — author describes as "quite breaking" |
| Common philosophy | Declarative, data-driven, hardware-isolated | Same |

### 6.2 Migration Path

If the convergence engine ships, the vendor functions defined here map directly into profile
modules:

| This proposal | Convergence Engine equivalent |
|---|---|
| `vendors/lenovo.rs` function | `profiles/lenovo.rs` module |
| `VendorRequirements` struct fields | Operation definitions with guards and effects |
| `boot_order_behavior: LenovoCleanup` | `op!(clean_boot_order { guard: ..., steps: [...] })` |
| `restart_protocol: IpmiRequired` | `op!(viking_restart { guard: contains(HwSku, "DGX"), steps: [action(ipmi_restart)] })` |
| Centralized execution engine | `ConvergenceEngine` scheduler |
| TOML `requirements` override | Config-assembled `state_desired` |

This proposal is the bridge — it isolates vendor behavior now and produces the exact inventory of
operations that would need to be defined in Phase 1 of the convergence engine migration.

### 6.3 Open Compilation Concern for PR #846

This proposal explicitly avoids a trait-based approach because the `async_trait` macro already
imposes ~3 seconds of compilation overhead per new platform in `libredfish`. The `op!` macro in
the convergence engine, depending on implementation, could have similar expansion costs per
operation per profile. This is worth raising in the PR #846 discussion as a design consideration
before the macro API is finalized.

---

## 7. Testing Strategy

The struct model enables two independent test layers.

### Layer 1 — Platform Requirements Tests

Pure unit tests: call the vendor function, assert struct fields. No async, no mock BMC, no
state machine.

```rust
#[test]
fn lenovo_does_not_support_bmc_lockdown() {
    let reqs = lenovo::requirements("ThinkSystem SR670 V2");
    assert!(!reqs.supports_bmc_lockdown);
}

#[test]
fn r760_disables_psu_hot_spare() {
    let reqs = dell::requirements("PowerEdge R760");
    let attr = reqs.preingestion_attributes.iter()
        .find(|a| a.path.contains("Attributes"))
        .expect("expected a Redfish attribute action");
    assert_eq!(attr.attributes["ServerPwr.1.PSRapidOn"], "Disabled");
}

#[test]
fn r750_has_no_psu_action() {
    // Confirm the R760 fix does not bleed into other Dell models
    let reqs = dell::requirements("PowerEdge R750");
    assert!(reqs.preingestion_attributes.is_empty());
}
```

### Layer 2 — Execution Engine Tests

Construct a `VendorRequirements` literal directly — no vendor function involved. The two layers
are fully decoupled.

```rust
#[tokio::test]
async fn bmc_reset_called_when_required_after_bmc_upgrade() {
    let reqs = VendorRequirements {
        needs_bmc_reset_after_bmc_upgrade: true,
        ..Default::default()
    };
    let mock = MockRedfishClient::new();
    execute_post_upgrade_actions(&reqs, UpgradeType::Bmc, &mock).await.unwrap();
    assert!(mock.bmc_reset_was_called());
}

#[tokio::test]
async fn optional_attribute_failure_does_not_fail_preingestion() {
    let reqs = VendorRequirements {
        preingestion_attributes: vec![RedfishAttributeSet {
            path: "/redfish/v1/Managers/System.Embedded.1/Attributes".into(),
            attributes: [("ServerPwr.1.PSRapidOn".into(), "Disabled".into())].into(),
            required: false,
        }],
        ..Default::default()
    };
    let mock = MockRedfishClient::new_that_fails_patches();
    assert!(execute_initial_actions(&reqs, &mock).await.is_ok());
}
```

### Pairing with the Supported Hardware List

If a supported hardware list is adopted (per the suggestion in the Slack thread), contributors
can write Layer 1 tests against `bmc-mock` to validate their declared requirements. Hardware
stays on the list as long as its tests pass — no physical hardware required.

---

## 8. Migration Plan

### Phase 1 — VendorRequirements Infrastructure

- Define `VendorRequirements` struct with `Default` and `Deserialize`
- Create `vendors/` directory with `dell.rs`, `lenovo.rs`, `nvidia.rs`
- Implement centralized execution engine in `preingestion_manager/mod.rs` and `handlers/`
- Migrate all Category A and B workarounds to struct fields
- Add TOML deserialization support in `host_models` config
- Full unit test coverage for both layers
- Add `CODEOWNERS` entries on `bmc-explorer` and `preingestion_manager`

### Short-Term (Gigabyte unblocked now)

Before Phase 1 lands, Krish's model-gating fix: add `HwType::GigabyteAmi` with `EndlessBoot`
disabled and a `BMCVendor` that is not `Nvidia`. This is safe, isolated, and reviewable. The
`BMCVendor::Nvidia` misclassification (via AMI TLS issuer mapping) is a separate but related
cleanup that should accompany this.

### Phase 2 — Deferred Cases

- Add `RestartProtocol::IpmiRequired` execution branch
- Migrate Viking IPMI restart from `redfish.rs` to `nvidia.rs` + execution engine
- At this point all known workarounds are in the struct-based system

### Phase 3 — Config Migration (Ongoing)

- Migrate vendor Rust functions to TOML config entries model by model as hardware is validated
- New hardware quirks: config entry first, Rust fallback if config is insufficient
- Policy: any new `if vendor.is_*()` in orchestration code requires a corresponding struct field
  and a linked issue reference

---

## 9. Open Questions

1. **Model inheritance / grouping.** If Dell releases an R760xa with the same PSU quirk, does the
   operator copy-paste the config? Mitigation: model pattern matching (contains / regex) rather
   than exact match. The vendor function already does this via `str::contains`.

2. **Config ownership.** Is `carbide.toml` operator-owned (per-site) or engineer-maintained
   (shipped with the software)? The hybrid narrows config surface to Category A attributes and
   requirements struct fields — a small, well-documented set.

3. **Workaround lifecycle.** Hardware workarounds have a lifecycle. As firmware matures, some
   requirements fields should return to default. Each non-default field in a vendor file should
   reference the issue that introduced it, making stale workarounds auditable.

4. **Category C deferral.** The Viking restart case is deferred to Phase 2. The struct field and
   execution branch should be added in Phase 1 even if `nvidia.rs` continues to return
   `StandardRedfish` until Phase 2.

5. **`BMCVendor` enum cleanup.** The `BMCVendor::Nvidia` catch-all for all AMI TLS issuers is
   independent of but related to this work. An `Ami` variant that is not `Nvidia` would prevent
   the Gigabyte misclassification class of bugs permanently.
