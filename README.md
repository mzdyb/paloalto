# Ansible automation for Palo Alto firewalls

This demo project contains Ansible playbooks for automating Palo Alto firewall operations, including:

- Firewall configuration
- Configuration drift detection
- Configuration drift remediation

It uses the `paloaltonetworks.panos` Ansible collection and a custom Execution Environment (EE).

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
Playbooks run inside a container-based Execution Environment built with `ansible-builder`. The EE packages all required collections and Python dependencies.

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

Firewall engineers can make configuration changes directly on firewalls through the GUI, such as emergency fixes, troubleshooting rules or manual tweaks. As a result, over time, the live firewall configuration can diverge from the desired state defined in Ansible variable files. This configuration drift creates security risks and makes the YAML files unreliable as a source of truth.

#### Two Types of Drift

The `detect_drift.yml` playbook identifies two distinct types of drift:

**1. Modification Drift** - configuration items defined in YAML have been changed from the firewall GUI (not in YAML which is source of truth)

Example: An interface IP address was changed from `10.0.1.1/24` to `10.0.1.5/24` directly on the firewall.

Detection method: Ansible `check_mode` with `diff` mode. Each configuration module runs in check mode, comparing the desired state (from YAML) against the live state. If `changed: true`, drift exists for that item.

**2. Unauthorized Additions** - New items have been added from the firewall GUI, these items are not defined in the YAML.

Example: An engineer added a temporary "Allow Vendor Access" security rule directly on the firewall.

Detection method: The `state: gathered` parameter retrieves all live items from the firewall. The Jinja2 `difference` filter compares gathered items against the YAML-defined items to find anything extra.

Combining both types of drift into a single `state: gathered` method would require writing custom Jinja2 dict comparison logic for each resource type and filtering out extra fields that the firewall returns but are not defined in the YAML. So in this project two detection methods are used to make the solution simpler.

**Known limitation:** The `panos_interface` module has a bug where `check_mode` always reports `changed: true` regardless of actual state. Interface drift results are shown in the report as informational but are excluded from the `drift_detected` flag. Support ticket has been opened for this module issue at the time of creating this project.

### Configuration Drift Remediation

The `remediate_drift.yml` playbook restores the firewall to the exact state defined in the YAML source of truth. It handles both types of drift: it corrects modifications and removes unauthorized additions.


## AAP Integration

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


## Author

[Michal Zdyb](https://www.linkedin.com/in/michal-zdyb-9aa4046/)