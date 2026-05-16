# Ansible automation for Palo Alto firewalls

This demo project contains Ansible playbooks for automating Palo Alto firewall operations, including:

- Firewall configuration
- Configuration drift detection
- Configuration drift remediation

It uses the `paloaltonetworks.panos` Ansible collection and a custom Execution Environment.

The project is designed for Red Hat Ansible Automation Platform (AAP), where it can use platform capabilities such as Automation Workflows, Custom Credentials, Schedulers and Event-Driven Automation. It can also be run from the command line using `ansible-navigator`.


## Quick start

### Firewall credentials

Provide credentials using one of these methods:

   ```bash
   # Option A: Command-line extra vars
   ansible-navigator run playbooks/configure_fw_network.yml \
     -e panos_username=admin -e panos_password=yourpassword

   # Option B: Ansible Vault (create vault file first)
   ansible-navigator run playbooks/configure_fw_network.yml \
     -e @vault.yml --vault-password-file .vault_pass

   # Option C: AAP Custom Credentials (injected automatically)
   For Option C details see "AAP Integration" section below.
   ```

### Building the EE
Playbooks run inside a container-based Execution Environment (EE) built with `ansible-builder`. The EE packages all required collections and Python dependencies.

```bash
cd execution-environment/
ansible-builder build -t aap.rh.lab/ee-panos:1.0 -v3
```

## Use cases

### Firewall configuration

1. **Apply configuration:**

   ```bash
   ansible-navigator run playbooks/configure_fw_network.yml
   ansible-navigator run playbooks/configure_fw_security_policies.yml
   ansible-navigator run playbooks/commit_fw_config.yml
   ```

2. **Verify configuration:**

   ```bash
   ansible-navigator run playbooks/show_fw_info.yml
   ```

### Configuration Drift Detection

#### The Problem

Firewall engineers routinely make out-of-band changes directly on the firewall (emergency fixes, troubleshooting rules, manual tweaks). Over time, the live firewall configuration diverges from the desired state defined in the Ansible variable files. This "configuration drift" creates security risks and makes the YAML files unreliable as a source of truth.

#### Two Types of Drift

The `detect_drift.yml` playbook identifies two distinct types of drift:

**1. Modification Drift** - configuration items defined in YAML have been changed on the firewall (not in YAML)

Example: An interface IP address was changed from `10.0.1.1/24` to `10.0.1.5/24` directly on the firewall.

Detection method: Ansible `check_mode` with `diff` mode. Each configuration module runs in check mode, comparing the desired state (from YAML) against the live state. If `changed: true`, drift exists for that item.

**2. Unauthorized Additions** - New items have been added on the firewall, these items are not defined in the YAML source of truth.

Example: An engineer added a temporary "Allow Vendor Access" security rule directly on the firewall.

Detection method: The `state: gathered` parameter retrieves all live items from the firewall. The Jinja2 `difference` filter compares gathered items against the YAML-defined items to find anything extra.

#### How Detection Works

The playbook runs in three phases:

**Phase 1 - Modification detection via `check_mode`**

Runs every configuration module in `check_mode: true` against the desired state. Ansible compares each parameter (IP address, nexthop, action etc.) against the live firewall and reports `changed: true` if any value differs. This leverages each module's built-in idempotency logic - no custom comparison code needed.

**Phase 2 - Unauthorized addition detection via `state: gathered`**

Uses `state: gathered` to retrieve all live security rules, address objects, zones and static routes. Compares the gathered item names against the YAML source of truth using the `difference` filter to identify items that exist on the firewall but not in the YAML files.

**Phase 3 - Reporting and AAP stats**

Generates a drift report showing exactly what has drifted, sets summary flags (`drift_detected`, `modification_drift_found`, `unauthorized_additions_found`) and exports them via `set_stats` for AAP workflow conditional next steps.

#### Why Two Phases?

The `state: gathered` parameter returns full parameter data for each resource, not just names. For example, a gathered static route includes `destination`, `nexthop`, `metric`, `nexthop_type` and other fields. In theory, this data alone could detect both unauthorized additions **and** parameter modifications in a single pass.

However, this project uses two separate phases because:

- **Phase 1 (`check_mode`)** gets parameter-level comparison for free - each module's idempotency logic handles the comparison internally, with no custom Jinja2 required.
- **Phase 2 (`state: gathered`)** is used only for name-level comparison via the `difference` filter, which is simple (no custom Jinia2 logic required)

Combining both into a single `state: gathered` phase would require writing custom Jinja2 dict comparison logic for each resource type and filtering out extra fields that the firewall returns but are not defined in the YAML (such as `metric`, `nexthop_type`, `admin_dist`). Those extra fields would cause false positives unless explicitly excluded.

**Known limitation:** The `panos_interface` module has a bug where `check_mode` always reports `changed: true` regardless of actual state. Interface drift results are shown in the report as informational but are excluded from the `drift_detected` flag. Support ticket has been opened for this issue at the time of creating this project.

### Configuration Drift Remediation

The `remediate_drift.yml` playbook restores the firewall to the exact state defined in the YAML source of truth. It handles both types of drift: it corrects modifications and removes unauthorized additions.

#### Remediation Steps

The playbook executes six steps in sequence:

| Step | Action | Method |
|---|---|---|
| 1 | **Backup** current running config | Imports `backup_fw_config.yml` |
| 2 | **Re-apply network config** | Imports `configure_fw_network.yml` |
| 3 | **Re-apply security rules** | Imports `configure_fw_security_policies.yml` |
| 4 | **Remove unauthorized items** | Gathers live state, computes `difference`, deletes extras with `state: absent` |
| 5 | **Commit** all changes | Imports `commit_fw_config.yml` |
| 6 | **Verify** firewall reachable | Runs `show system info`, exports `remediation_completed: true` |

Steps 1-3 and 5 reuse existing playbooks via `import_playbook` to avoid code duplication. Step 4 is a dedicated play that gathers live configuration, computes which items are not in the YAML files and removes them.

## AAP Integration

### Job Templates

Create the following Job Templates in Ansible Automation Platform:

| Job Template | Playbook | Purpose |
|---|---|---|
| FW - Detect Drift | `playbooks/detect_drift.yml` | Scheduled drift detection |
| FW - Remediate Drift | `playbooks/remediate_drift.yml` | Manual remediation trigger |
| FW - Configure Network | `playbooks/configure_fw_network.yml` | Network provisioning |
| FW - Configure Security Policies | `playbooks/configure_fw_security_policies.yml` | Security rule provisioning |
| FW - Commit Config | `playbooks/commit_fw_config.yml` | Commit pending changes |
| FW - Show Info | `playbooks/show_fw_info.yml` | Operational data gathering |
| FW - Backup Config | `playbooks/backup_fw_config.yml` | Configuration backup |

### Workflow Job Templates

#### Workflow 1: Firewall Provisioning

Applies full configuration to a new or factory-reset firewall.

```
[Configure Network] ──▶ [Configure Security Policies] ──▶ [Commit Config] ──▶ [Show Info]
```

No extra vars needed — playbooks default to `state: present`. No backup step since this targets firewalls with no existing config to preserve. Show Info at the end provides verification output in the job log.

#### Workflow 2: Firewall Deprovisioning

Removes all managed configuration (e.g., decommissioning a firewall or resetting to clean state).

```
[Backup Config] ──▶ [Configure Security Policies] ──▶ [Configure Network] ──▶ [Commit Config]
                       (fw_rules_state=absent)          (fw_config_state=absent)
```

Order matters: security policies are removed before network config because rules reference zones and objects that network config creates. Set `fw_rules_state: absent` and `fw_config_state: absent` as extra vars on the respective workflow nodes.

#### Workflow 3: Day 2 Configuration Change

Safe change workflow for modifying existing firewall configuration. Engineer updates host_vars in Git, then triggers this workflow.

```
[Backup Config] ──▶ [Configure Network] ──▶ [Configure Security Policies] ──▶ [Commit Config] ──▶ [Detect Drift]
```

Backup captures the pre-change state. After commit, Detect Drift verifies that the firewall matches the source of truth (expected result: `drift_detected: false`).

#### Workflow 4: Drift Detection & Remediation

Scheduled workflow that detects drift and conditionally triggers remediation with human approval.

```
[Detect Drift] ──▶ drift_detected == true  ──▶ [Approval Node] ──▶ [Remediate Drift]
               │
               └── drift_detected == false ──▶ [No Action]
```

The conditional branching works because `detect_drift.yml` exports `drift_detected` via `set_stats`. AAP evaluates this variable to decide which branch to follow. Schedule nightly (e.g., 2:00 AM) or after maintenance windows. Attach a notification template (email/Slack) to the Approval Node so engineers are alerted when drift is found.

### Credential Configuration

Create an AAP **Custom Credential Type** for PAN-OS:

**Input Configuration:**
```yaml
fields:
  - id: panos_username
    type: string
    label: PAN-OS Username
  - id: panos_password
    type: string
    label: PAN-OS Password
    secret: true
```

**Injector Configuration:**
```yaml
extra_vars:
  panos_username: "{{ panos_username }}"
  panos_password: "{{ panos_password }}"
```





## Testing (section to be removed)

The project includes three layers of testing, each validating different aspects of the solution.

### Layer 1: Static Analysis

Linting with `ansible-lint` and `yamllint` to catch syntax errors, style violations and Ansible anti-patterns.

```bash
# YAML lint
yamllint -c .yamllint.yml .

# Ansible lint (run inside the EE to resolve all collections)
ansible-navigator exec -- ansible-lint playbooks/

# Ansible syntax check
ansible-navigator run playbooks/detect_drift.yml -- --syntax-check
ansible-navigator run playbooks/remediate_drift.yml -- --syntax-check
ansible-navigator run playbooks/configure_fw_network.yml -- --syntax-check
ansible-navigator run playbooks/configure_fw_security_policies.yml -- --syntax-check
```

### Layer 2: Unit Tests

Mock-based tests that validate the drift detection logic without connecting to a firewall. These tests run on `localhost` using simulated `check_mode` results and `state: gathered` data to verify that the Jinja2 expressions correctly classify drift.

```bash
ansible-navigator run tests/unit/test_drift_logic.yml
```

**Test scenarios covered:**

| Scenario | What it validates |
|---|---|
| Clean state (no drift) | All flags are `false` when no changes detected |
| Modification drift only | `modification_drift_found` is `true`, unauthorized flags are `false` |
| Unauthorized additions only | `unauthorized_additions_found` is `true`, modification flags are `false` |
| Both drift types simultaneously | Both flags are `true`, `drift_detected` is `true` |
| Empty gathered results | `default([])` fallbacks work correctly |

### Layer 3: Integration Tests

End-to-end tests that run against a live PAN-OS firewall. These tests require a firewall configured to match the host_vars source of truth.

```bash
# Test 1: Verify baseline - no drift on a correctly configured firewall
ansible-navigator run tests/integration/test_detect_drift.yml \
  -e panos_username=admin -e panos_password=yourpassword

# Test 2: Full cycle - inject drift, detect it, remediate, verify clean
ansible-navigator run tests/integration/test_drift_and_remediate.yml \
  -e panos_username=admin -e panos_password=yourpassword
```

**Full cycle test phases:**

1. **Inject drift** - Creates a rogue address object and security rule on the firewall
2. **Detect** - Runs `detect_drift.yml` and asserts the rogue items are identified
3. **Remediate** - Runs `remediate_drift.yml` to restore desired state
4. **Verify** - Runs `detect_drift.yml` again and asserts no remaining drift
