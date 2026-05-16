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

This project follows a Configuration-as-Code approach: the firewall configuration is defined declaratively in YAML variable files (`host_vars/fw1/`) and applied through Ansible playbooks. Instead of making changes manually through the firewall GUI, engineers modify version-controlled YAML files and run playbooks to push the desired state to the firewall.

This approach provides several benefits. All changes are tracked in Git with a full audit trail and deployments are repeatable and consistent across firewalls. It also allows the YAML files to serve as the source of truth for firewall configuration and provides a foundation for drift detection and remediation.


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

The following automation workflows have been created on AAP.

### Workflow Job Templates

#### Workflow 1: Firewall Provisioning

Applies full configuration to a new or factory-reset firewall  
![provision firewall](<files/paloalto - provision firewall.png>)

#### Workflow 2: Firewall Deprovisioning

Removes all the provisioned configuration  
![deprovision firewall](<files/paloalto - deprovision firewall.png>)

Extra vars `fw_rules_state: absent` and `fw_config_state: absent` are set on the respective workflow nodes.

#### Workflow 3: Day 2 Configuration Change

Modifies existing firewall configuration  
![day 2 configuration change](<files/paloalto - day 2 configuration change.png>)


#### Workflow 4: Drift Detection & Remediation

Detects drift and conditionally triggers remediation with human approval  
![drift detection and remediation](<files/paloalto - drift detection and remediation.png>)

It can be scheduled to run periodically.
To do: slack notification playbook and node.

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

## AAP Configuration as Code

The AAP Workflows used in this project are managed using a Configuration as Code approach. Their configuration is stored in a separate repository:  
https://github.com/mzdyb/aap-configuration-as-code

## Author

[@mzdyb](https://www.linkedin.com/in/michal-zdyb-9aa4046/)