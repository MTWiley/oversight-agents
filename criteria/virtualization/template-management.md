# VM Template and Golden Image Management

Reference checklist for the `virtualization-reviewer` agent when evaluating VM template creation, lifecycle management, customization specifications, golden image pipelines, and Packer template best practices across VMware vSphere, Proxmox VE, and KVM/libvirt environments. This file complements the inline criteria in `review-virtualization.md` with detailed checkpoints, severity classifications, platform-specific examples, and remediation guidance.

Covers: template creation and hardening, guest tools installation, cloud-init/sysprep configuration, credential and SSH key cleanup, template versioning and lifecycle, deprecation workflows, automated template creation with Packer, customization specifications, unique identifier handling, golden image CI/CD pipelines, compliance scanning, and Packer best practices.

---

## 1. Template Creation and Base OS Hardening

**Category**: Template Security

VM templates are the foundation for every deployed VM. A poorly secured template propagates vulnerabilities to every VM instantiated from it. Templates must start with a minimal OS installation, apply hardening baselines, install required agents, and remove all unique identifiers and credentials before conversion to template.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 1.1 | Template uses a minimal OS installation (server core, minimal install, no GUI unless required) | **MEDIUM** | All |
| 1.2 | OS is hardened according to an applicable CIS benchmark or organizational hardening standard | **HIGH** | All |
| 1.3 | Unnecessary services are disabled (Bluetooth, printing, CUPS, Avahi, telnet, FTP server) | **MEDIUM** | All |
| 1.4 | Unnecessary packages are removed to minimize attack surface | **MEDIUM** | All |
| 1.5 | Firewall is enabled with a default-deny inbound policy | **HIGH** | All |
| 1.6 | SSH is configured securely (key-based auth, no root login, strong ciphers) | **HIGH** | Linux |
| 1.7 | Password policies are configured (complexity, expiration, lockout) | **MEDIUM** | All |
| 1.8 | Automatic security updates are enabled or a documented patching mechanism is defined | **HIGH** | All |
| 1.9 | Audit logging (auditd, Windows Event Logging) is configured | **MEDIUM** | All |
| 1.10 | SELinux/AppArmor is in enforcing mode (Linux templates) | **HIGH** | Linux |
| 1.11 | Local administrator/root password is set to a known-strong value or locked (to be reset by customization) | **HIGH** | All |
| 1.12 | Template includes only the partitioning scheme required for the workload (separate /var, /tmp, /home where appropriate) | **MEDIUM** | Linux |
| 1.13 | Disk alignment is correct for the underlying storage (4K sector alignment) | **LOW** | All |
| 1.14 | UEFI boot with Secure Boot is enabled where the OS supports it | **MEDIUM** | All |
| 1.15 | Template VM hardware version matches or is one below the cluster maximum (for compatibility) | **LOW** | vSphere |

### Common Mistakes

**Mistake**: Template created from a full desktop OS installation with GUI, multimedia codecs, and development tools.

Why it is a problem: Every unnecessary package is a potential vulnerability. A full GNOME desktop on a server template includes hundreds of packages that will never be used but must be patched. This increases the attack surface, patch cycle time, and template storage footprint.

```bash
# BAD - full desktop installation for a server template
# RHEL/CentOS: "Server with GUI" or "Workstation" group installed

# GOOD - minimal installation
# RHEL/CentOS: "Minimal Install" group
# Ubuntu: ubuntu-server (minimal), no ubuntu-desktop
# Debian: standard system utilities only, no desktop environment
```

**Mistake**: Template without CIS benchmark hardening.

Why it is a problem: Default OS installations are configured for ease of use, not security. Default configurations include: open firewall, weak password policy, unnecessary services running, permissive file permissions, and unpatched software. CIS benchmarks provide a systematically reviewed and tested hardening checklist.

```bash
# Apply CIS hardening via automation
# RHEL/CentOS - using OSCAP (OpenSCAP)
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis \
    --remediate /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Ubuntu - using CIS benchmark Ansible role
ansible-playbook cis-ubuntu-hardening.yml --tags level1-server
```

---

## 2. Guest Tools and Agent Installation

**Category**: Template Configuration

Guest tools (VMware Tools, QEMU Guest Agent, Proxmox QEMU Guest Agent) provide critical functionality: graceful shutdown, time synchronization, heartbeat monitoring, memory management (ballooning), and customization support. Missing guest tools degrade the hypervisor's ability to manage the VM and prevent proper customization during deployment.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 2.1 | VMware Tools (open-vm-tools) is installed and current on vSphere templates | **HIGH** | vSphere |
| 2.2 | QEMU Guest Agent is installed on Proxmox and KVM templates | **HIGH** | Proxmox, KVM |
| 2.3 | Guest tools service is enabled to start at boot | **MEDIUM** | All |
| 2.4 | Guest tools version is compatible with the target hypervisor version | **MEDIUM** | All |
| 2.5 | Cloud-init is installed on Linux templates for customization | **HIGH** | All |
| 2.6 | Cloudbase-init is installed on Windows templates for cloud/customization support | **MEDIUM** | All |
| 2.7 | Required monitoring agents are pre-installed (but not pre-configured with environment-specific settings) | **LOW** | All |
| 2.8 | Configuration management agent (Ansible target deps, Puppet, Chef) is pre-installed if used in the environment | **LOW** | All |
| 2.9 | Virtio drivers are installed on Windows templates for KVM/Proxmox use | **HIGH** | KVM, Proxmox |

### Common Mistakes

**Mistake**: Template without open-vm-tools, relying on bundled VMware Tools ISO.

Why it is a problem: Without VMware Tools, vSphere cannot perform graceful shutdown, customization (sysprep/cloud-init), or guest heartbeat monitoring. HA may incorrectly report the VM as failed if guest heartbeats are used. open-vm-tools is maintained as part of the Linux distribution and updated via normal package management, unlike the bundled VMware Tools which requires manual ISO-based installation.

```bash
# Linux template - install open-vm-tools
# RHEL/CentOS
dnf install -y open-vm-tools
systemctl enable vmtoolsd

# Ubuntu/Debian
apt-get install -y open-vm-tools
systemctl enable open-vm-tools

# For desktop VMs, also install open-vm-tools-desktop
```

**Mistake**: Windows KVM template without virtio drivers.

Why it is a problem: Without virtio drivers, Windows VMs on KVM/Proxmox fall back to emulated IDE disks and Intel e1000 network adapters, resulting in dramatically worse disk and network performance. Install the virtio-win drivers during template creation by attaching the virtio-win ISO.

```powershell
# Windows template - install virtio drivers from ISO
# Mount virtio-win ISO and install:
# - viostor (SCSI) driver for virtio-scsi disk
# - netkvm (network) driver for virtio-net
# - balloon (memory) driver for memory ballooning
# - qemu-ga (guest agent) for QEMU Guest Agent

# PowerShell - install QEMU Guest Agent from virtio-win ISO
msiexec /i E:\guest-agent\qemu-ga-x86_64.msi /quiet /norestart
```

---

## 3. Cloud-init and Sysprep Configuration

**Category**: Customization

Cloud-init (Linux) and sysprep (Windows) are the primary mechanisms for customizing VMs deployed from templates. They handle hostname assignment, network configuration, user creation, SSH key injection, and domain join. Incorrect configuration leads to duplicate identities, failed customizations, and deployment errors.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 3.1 | Cloud-init is installed and the appropriate datasource is configured for the hypervisor | **HIGH** | Linux |
| 3.2 | Cloud-init `NoCloud`, `VMware`, or `ConfigDrive` datasource is configured for on-premises deployments | **MEDIUM** | Linux |
| 3.3 | Cloud-init configuration resets hostname, machine-id, SSH host keys, and network config on first boot | **HIGH** | Linux |
| 3.4 | Cloud-init `manage_etc_hosts` is configured to update /etc/hosts with the new hostname | **LOW** | Linux |
| 3.5 | Sysprep answer file (unattend.xml) is tested and produces unique SIDs, hostnames, and network configuration | **HIGH** | Windows |
| 3.6 | Sysprep answer file does not contain plaintext passwords or product keys | **CRITICAL** | Windows |
| 3.7 | Sysprep answer file includes the correct Windows edition and locale settings | **MEDIUM** | Windows |
| 3.8 | vSphere customization specifications are created and tested for each OS template | **MEDIUM** | vSphere |
| 3.9 | Customization specification includes DNS, domain, and network settings appropriate to the target environment | **MEDIUM** | vSphere |
| 3.10 | Customization specification includes domain join credentials stored securely (not in plaintext in vCenter) | **HIGH** | vSphere |
| 3.11 | Proxmox cloud-init configuration provides IP, gateway, DNS, and SSH key settings via GUI or API | **MEDIUM** | Proxmox |
| 3.12 | Cloud-init `final_message` module is configured to signal completion of customization | **LOW** | All |

### Common Mistakes

**Mistake**: Cloud-init installed but `/etc/machine-id` not truncated before templating.

Why it is a problem: `/etc/machine-id` is used by systemd, DHCP clients, and various services as a unique identifier. If it is not emptied before converting to template, every VM deployed from the template has the same machine-id. This causes DHCP lease conflicts (all VMs get the same IP), systemd journal conflicts, and dbus identifier collisions.

```bash
# Before converting to template, clean machine-id
# Method 1: Truncate (cloud-init will regenerate on first boot)
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id

# Method 2: Use cloud-init clean (preferred)
cloud-init clean --logs --machine-id --seed
```

**Mistake**: Sysprep answer file containing plaintext administrator password.

```xml
<!-- BAD - plaintext password in unattend.xml -->
<UserAccounts>
    <AdministratorPassword>
        <Value>P@ssw0rd123!</Value>
        <PlainText>true</PlainText>
    </AdministratorPassword>
</UserAccounts>

<!-- GOOD - password set via customization spec or post-deployment automation -->
<!-- Or if password must be in unattend.xml, use encoded form and rotate immediately -->
<UserAccounts>
    <AdministratorPassword>
        <Value>UABAAHMAcwB3ADAAcgBkADEAMgAzACEA...</Value>
        <PlainText>false</PlainText>
    </AdministratorPassword>
</UserAccounts>
<!-- NOTE: Base64 encoding is NOT encryption. Use post-deployment credential rotation. -->
```

**Mistake**: vSphere customization spec with domain join credentials visible to all vCenter administrators.

Why it is a problem: vSphere customization specifications store domain join credentials in vCenter. Any user with access to view or export the customization spec can extract the domain join account password. Use a dedicated service account with minimal privileges (only "join computer to domain") and rotate the password regularly.

---

## 4. Credential and SSH Key Cleanup

**Category**: Template Security

Before converting a VM to a template, all credentials, SSH keys, and session data must be removed. Templates are shared broadly and stored on shared datastores. Any credentials left in the template are accessible to everyone who can deploy from it.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 4.1 | All user SSH authorized_keys files are emptied | **CRITICAL** | Linux |
| 4.2 | SSH host keys are removed (cloud-init regenerates them on first boot) | **HIGH** | Linux |
| 4.3 | Shell history is cleared for all users (root, service accounts, builder accounts) | **HIGH** | Linux |
| 4.4 | Temporary files and caches are cleared (/tmp, /var/tmp, apt/yum cache) | **LOW** | Linux |
| 4.5 | Log files are truncated (not deleted, to preserve file permissions and SELinux contexts) | **MEDIUM** | Linux |
| 4.6 | Builder/packer user accounts are removed from the template | **HIGH** | All |
| 4.7 | Sudo NOPASSWD entries for builder accounts are removed | **CRITICAL** | Linux |
| 4.8 | Any API tokens, license keys, or temporary credentials used during build are removed | **CRITICAL** | All |
| 4.9 | Network configuration is reset to DHCP or removed (customization will set it) | **MEDIUM** | All |
| 4.10 | Persistent network interface names (udev rules) are removed to prevent NIC naming issues on deployment | **MEDIUM** | Linux |
| 4.11 | Windows user profiles for builder accounts are removed | **HIGH** | Windows |
| 4.12 | Windows credential manager entries are cleared | **HIGH** | Windows |
| 4.13 | Browser history, saved passwords, and cached credentials are removed | **MEDIUM** | All |
| 4.14 | `/etc/machine-id` is truncated (see Section 3) | **HIGH** | Linux |
| 4.15 | Cloud-init state is cleaned so it runs fresh on first boot | **HIGH** | Linux |

### What to Look For

```bash
# Linux template cleanup script (run before converting to template)
#!/bin/bash
set -euo pipefail

# Remove SSH host keys (cloud-init regenerates)
rm -f /etc/ssh/ssh_host_*

# Remove all authorized_keys
find / -name authorized_keys -exec truncate -s 0 {} \;

# Remove builder user if it exists
userdel -r packer 2>/dev/null || true
userdel -r builder 2>/dev/null || true

# Remove sudo NOPASSWD entries for builder
rm -f /etc/sudoers.d/packer /etc/sudoers.d/builder

# Clear shell history for all users
find /root /home -name ".bash_history" -exec truncate -s 0 {} \; 2>/dev/null
find /root /home -name ".zsh_history" -exec truncate -s 0 {} \; 2>/dev/null
history -c

# Clean package cache
dnf clean all 2>/dev/null || apt-get clean 2>/dev/null || true

# Truncate log files (preserve permissions and SELinux context)
find /var/log -type f -exec truncate -s 0 {} \;

# Remove temporary files
rm -rf /tmp/* /var/tmp/*

# Remove persistent network rules
rm -f /etc/udev/rules.d/70-persistent-net.rules
rm -f /etc/udev/rules.d/75-persistent-net-generator.rules

# Truncate machine-id
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id

# Clean cloud-init state
cloud-init clean --logs --machine-id --seed

# Zero free disk space for thin provisioning efficiency (optional, time-consuming)
# dd if=/dev/zero of=/EMPTY bs=1M 2>/dev/null; rm -f /EMPTY
```

### Common Mistakes

**Mistake**: SSH authorized_keys containing the Packer builder's public key left in the template.

Why it is a problem: The Packer builder key provides passwordless SSH access to every VM deployed from this template. If the corresponding private key is compromised (or is stored in a CI system with broad access), every VM is immediately accessible.

**Mistake**: Shell history containing passwords, API tokens, or sensitive commands.

```bash
# BAD - history reveals sensitive data
$ mysql -u root -pMySecretPassword
$ curl -H "Authorization: Bearer eyJhbGciOi..." https://api.example.com
$ export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCY...
# These are now in .bash_history in the template
```

**Mistake**: Sudo NOPASSWD entry for the Packer builder account left in the template.

```bash
# BAD - /etc/sudoers.d/packer still present in template
packer ALL=(ALL) NOPASSWD:ALL

# This gives passwordless root access to the 'packer' user.
# Even if the user is deleted, if the file remains and a new
# user named 'packer' is created, they get passwordless root.
```

---

## 5. Template Versioning and Lifecycle

**Category**: Template Lifecycle

Templates are living artifacts that must be updated, versioned, and eventually deprecated. Without a lifecycle process, templates accumulate vulnerabilities, drift from compliance baselines, and create inconsistency across the environment. Every template should have a clear owner, version, patch date, and deprecation policy.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 5.1 | Templates use a consistent naming convention including OS, version, date, and build number | **MEDIUM** | All |
| 5.2 | Template metadata (annotations, notes, tags) records OS version, patch level, build date, and CIS benchmark version | **MEDIUM** | All |
| 5.3 | Templates are rebuilt (not patched in-place) on a regular schedule (monthly recommended, quarterly minimum) | **HIGH** | All |
| 5.4 | Template update schedule aligns with OS vendor patch cycles (Patch Tuesday, RHEL errata) | **MEDIUM** | All |
| 5.5 | Old templates are deprecated with a documented sunset date (not silently deleted) | **MEDIUM** | All |
| 5.6 | Deprecated templates are marked (renamed, moved to a deprecated folder, permissions restricted) | **LOW** | All |
| 5.7 | Template inventory is maintained with current and deprecated versions | **MEDIUM** | All |
| 5.8 | Templates are stored on datastores/storage accessible to all deployment targets | **MEDIUM** | All |
| 5.9 | Content Library (vSphere) or equivalent is used for template distribution across sites | **MEDIUM** | vSphere |
| 5.10 | Template content library synchronization is verified for multi-site environments | **MEDIUM** | vSphere |
| 5.11 | Rollback to a previous template version is possible (previous versions retained for defined period) | **LOW** | All |

### Common Mistakes

**Mistake**: Template last updated 18 months ago with hundreds of missing security patches.

Why it is a problem: Every VM deployed from this template starts with 18 months of unpatched vulnerabilities. Even with automated patching post-deployment, there is a window between deployment and first patch cycle where the VM is vulnerable. Additionally, applying 18 months of patches at first boot significantly extends deployment time. Rebuild templates at least quarterly.

**Mistake**: Templates named without version or date information.

```
# BAD - no way to determine currency
Templates/
    rhel9-server
    ubuntu2204-server
    windows2022-standard

# GOOD - versioned with build date
Templates/
    rhel9-server-cis-v1.2-20260115
    rhel9-server-cis-v1.1-20241015    # Deprecated 2025-01-15
    ubuntu2204-server-cis-v1.3-20260201
    windows2022-std-cis-v1.0-20260110
```

**Mistake**: Patching a template in-place instead of rebuilding from scratch.

Why it is a problem: In-place patching accumulates configuration drift, leftover files from removed packages, and state changes from prior patches. A clean rebuild from the current OS media plus current patches produces a known-good, reproducible baseline. In-place patching also makes it impossible to verify that the hardening baseline is fully applied.

---

## 6. Automated Template Creation with Packer

**Category**: Automation

HashiCorp Packer automates VM template creation, ensuring templates are reproducible, consistent, and auditable. Manual template creation is error-prone, undocumented, and unrepeatable. Packer templates (HCL2 format) should be version-controlled, reviewed, and integrated into CI/CD pipelines.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 6.1 | Templates are created via Packer (or equivalent automation), not manually | **HIGH** | All |
| 6.2 | Packer templates use HCL2 format (not legacy JSON) | **LOW** | All |
| 6.3 | Packer template files are stored in version control | **HIGH** | All |
| 6.4 | Packer variables are used for environment-specific values (vCenter URL, datastore, network, ISO path) | **MEDIUM** | All |
| 6.5 | Packer sensitive variables (passwords, API tokens) are not hardcoded in template files | **CRITICAL** | All |
| 6.6 | Packer communicator uses SSH keys (not passwords) where possible | **MEDIUM** | All |
| 6.7 | Packer provisioners run hardening scripts, agent installation, and cleanup in a defined order | **MEDIUM** | All |
| 6.8 | Packer post-processors export the template to the target platform (vSphere template, Proxmox template, qcow2 image) | **MEDIUM** | All |
| 6.9 | Packer `boot_command` or `http_content` uses a kickstart/preseed/autounattend file for unattended installation | **MEDIUM** | All |
| 6.10 | Packer build timeout is configured to prevent zombie builds | **LOW** | All |
| 6.11 | Packer builds are idempotent (running the same build twice produces an equivalent result) | **MEDIUM** | All |
| 6.12 | Packer build artifacts (ISOs, logs) are cleaned up after successful builds | **LOW** | All |

### Example Correct Configuration

**Packer HCL2 template for vSphere (RHEL 9)**:

```hcl
packer {
  required_plugins {
    vsphere = {
      version = ">= 1.3.0"
      source  = "github.com/hashicorp/vsphere"
    }
  }
}

variable "vcenter_server" {
  type        = string
  description = "vCenter Server FQDN"
}

variable "vcenter_username" {
  type        = string
  description = "vCenter username"
}

variable "vcenter_password" {
  type      = string
  sensitive = true
}

variable "ssh_password" {
  type      = string
  sensitive = true
}

variable "build_version" {
  type    = string
  default = "1.0"
}

locals {
  build_date = formatdate("YYYYMMDD", timestamp())
  vm_name    = "rhel9-cis-v${var.build_version}-${local.build_date}"
}

source "vsphere-iso" "rhel9" {
  vcenter_server      = var.vcenter_server
  username            = var.vcenter_username
  password            = var.vcenter_password
  insecure_connection = false

  datacenter   = "DC-01"
  cluster      = "Build-Cluster"
  datastore    = "ds-templates"
  folder       = "Templates/Linux"
  resource_pool = "Build-Pool"

  vm_name              = local.vm_name
  guest_os_type        = "rhel9_64Guest"
  CPUs                 = 2
  RAM                  = 4096
  RAM_reserve_all      = false
  disk_controller_type = ["pvscsi"]
  firmware             = "efi-secure"

  storage {
    disk_size             = 40960
    disk_thin_provisioned = true
  }

  network_adapters {
    network      = "Build-Network"
    network_card = "vmxnet3"
  }

  iso_paths = [
    "[ds-isos] ISOs/rhel-9.4-x86_64-dvd.iso"
  ]

  boot_command = [
    "<up><wait>",
    "e<wait>",
    "<down><down><end>",
    " inst.ks=http://{{ .HTTPIP }}:{{ .HTTPPort }}/ks.cfg",
    "<F10>"
  ]

  http_content = {
    "/ks.cfg" = templatefile("${path.root}/http/ks.cfg.pkrtpl", {
      ssh_password = var.ssh_password
    })
  }

  ssh_username         = "root"
  ssh_password         = var.ssh_password
  ssh_timeout          = "30m"
  ssh_handshake_attempts = 100

  shutdown_command = "echo '${var.ssh_password}' | sudo -S shutdown -P now"

  convert_to_template = true
  notes               = "RHEL 9 | CIS L1 | Built ${local.build_date} | v${var.build_version}"
}

build {
  sources = ["source.vsphere-iso.rhel9"]

  # Phase 1: System updates
  provisioner "shell" {
    inline = [
      "dnf update -y",
      "dnf install -y open-vm-tools cloud-init",
      "systemctl enable vmtoolsd cloud-init"
    ]
  }

  # Phase 2: CIS hardening
  provisioner "ansible" {
    playbook_file = "${path.root}/ansible/cis-hardening.yml"
    extra_arguments = [
      "--extra-vars", "cis_level=1"
    ]
  }

  # Phase 3: Cleanup
  provisioner "shell" {
    script = "${path.root}/scripts/cleanup.sh"
  }
}
```

**Packer HCL2 template for Proxmox (Ubuntu 22.04)**:

```hcl
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.1.0"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

variable "proxmox_url" {
  type = string
}

variable "proxmox_token_id" {
  type = string
}

variable "proxmox_token_secret" {
  type      = string
  sensitive = true
}

source "proxmox-iso" "ubuntu2204" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_token_id
  token                    = var.proxmox_token_secret
  insecure_skip_tls_verify = false

  node     = "pve1"
  vm_name  = "ubuntu2204-template"
  vm_id    = 9000
  template_name        = "ubuntu2204-template"
  template_description = "Ubuntu 22.04 | CIS L1 | ${formatdate("YYYY-MM-DD", timestamp())}"

  os       = "l26"
  cpu_type = "host"
  cores    = 2
  memory   = 4096

  scsi_controller = "virtio-scsi-single"

  disks {
    type              = "scsi"
    disk_size         = "40G"
    storage_pool      = "local-lvm"
    format            = "raw"
  }

  network_adapters {
    model    = "virtio"
    bridge   = "vmbr0"
    vlan_tag = "100"
  }

  iso_file = "local:iso/ubuntu-22.04.4-live-server-amd64.iso"

  boot_command = [
    "c<wait>",
    "linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/'",
    "<enter><wait>",
    "initrd /casper/initrd",
    "<enter><wait>",
    "boot<enter>"
  ]

  http_directory = "${path.root}/http"

  ssh_username = "packer"
  ssh_private_key_file = "~/.ssh/packer_ed25519"
  ssh_timeout  = "30m"

  cloud_init              = true
  cloud_init_storage_pool = "local-lvm"
}

build {
  sources = ["source.proxmox-iso.ubuntu2204"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get upgrade -y",
      "sudo apt-get install -y qemu-guest-agent cloud-init",
      "sudo systemctl enable qemu-guest-agent"
    ]
  }

  provisioner "ansible" {
    playbook_file = "${path.root}/ansible/cis-hardening.yml"
  }

  provisioner "shell" {
    script = "${path.root}/scripts/cleanup.sh"
  }
}
```

### Common Mistakes

**Mistake**: Packer template with hardcoded vCenter password.

```hcl
# BAD - password in version control
variable "vcenter_password" {
  default = "VMware1!"
}

# GOOD - sensitive variable with no default, injected at build time
variable "vcenter_password" {
  type      = string
  sensitive = true
  # No default - must be provided via:
  # - PKR_VAR_vcenter_password environment variable
  # - -var flag
  # - .auto.pkrvars.hcl (gitignored)
  # - HashiCorp Vault
}
```

**Mistake**: Packer build using `insecure_connection = true` in production.

```hcl
# BAD - disables TLS certificate verification
insecure_connection = true

# GOOD - trust the CA certificate
insecure_connection = false
# Ensure the vCenter CA certificate is trusted on the build machine
```

---

## 7. Customization Specifications and Unique Identifiers

**Category**: Identity Management

Every VM deployed from a template must have unique identifiers: hostname, machine-id, SID (Windows), SSH host keys, MAC addresses (assigned by hypervisor), and DHCP client IDs. Duplicate identifiers cause network conflicts, authentication failures, domain join problems, and monitoring confusion.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 7.1 | Deployed VMs receive unique hostnames via customization spec, cloud-init, or naming convention | **HIGH** | All |
| 7.2 | Windows VMs receive unique SIDs via sysprep during deployment | **HIGH** | Windows |
| 7.3 | Linux VMs regenerate `/etc/machine-id` on first boot (cloud-init or systemd-firstboot) | **HIGH** | Linux |
| 7.4 | SSH host keys are regenerated on first boot (not shared across VMs from the same template) | **CRITICAL** | Linux |
| 7.5 | Network configuration (IP, subnet, gateway, DNS) is applied via customization, not baked into template | **HIGH** | All |
| 7.6 | Domain join occurs during customization with the correct OU, security group, and permissions | **MEDIUM** | Windows |
| 7.7 | DNS registration (forward and reverse) is handled during or immediately after deployment | **MEDIUM** | All |
| 7.8 | DHCP client ID is unique per VM (automatically handled by unique MAC address) | **LOW** | All |
| 7.9 | vSphere customization spec is tested by deploying a VM and verifying all unique identifiers | **MEDIUM** | vSphere |
| 7.10 | Deployment automation verifies customization completed successfully (check VMware Tools customization status or cloud-init result) | **MEDIUM** | All |

### Common Mistakes

**Mistake**: SSH host keys not removed from template, causing all deployed VMs to share the same host keys.

Why it is a problem: Shared SSH host keys mean an attacker who compromises one VM's SSH key (via backup exposure, template access, or similar) can impersonate any VM deployed from the same template. SSH host key verification warnings will not trigger because all VMs present the same key. This completely undermines SSH host authentication.

```bash
# Verify SSH host keys are unique after deployment
# On two VMs from the same template:
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# The fingerprints MUST be different. If they are the same,
# the template was not properly cleaned.
```

**Mistake**: Windows VMs deployed without sysprep, sharing the same SID.

Why it is a problem: Duplicate SIDs cause authentication failures in domain environments, security token conflicts, and unpredictable behavior with security-sensitive software. While Microsoft has stated that duplicate SIDs on domain-joined machines are handled by the domain SID, security software, licensing tools, and some applications still rely on machine SID uniqueness.

---

## 8. Golden Image CI/CD Pipeline

**Category**: Image Pipeline

A golden image pipeline automates the build, test, compliance scan, and promotion of VM templates through a CI/CD process. This ensures templates are always current, tested, and compliant before deployment. Without a pipeline, template updates are ad-hoc, untested, and inconsistent.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 8.1 | Template creation is triggered by CI/CD (not manual Packer runs on developer workstations) | **HIGH** | All |
| 8.2 | Pipeline triggers include: scheduled (weekly/monthly), OS vendor patch release, and manual override | **MEDIUM** | All |
| 8.3 | Pipeline stages include: build, test, compliance scan, promote, and cleanup | **HIGH** | All |
| 8.4 | Testing stage deploys a VM from the template and runs validation tests | **HIGH** | All |
| 8.5 | Validation tests verify: boot success, agent health (guest tools, cloud-init), network connectivity, customization success | **HIGH** | All |
| 8.6 | Compliance scanning runs CIS benchmark validation (OpenSCAP, InSpec, or equivalent) on the built image | **HIGH** | All |
| 8.7 | Vulnerability scanning runs on the template image (Trivy, Grype, Qualys, Tenable) | **HIGH** | All |
| 8.8 | Pipeline fails if compliance score is below threshold or critical vulnerabilities are found | **HIGH** | All |
| 8.9 | Successful builds are promoted to production template storage (content library, template folder) | **MEDIUM** | All |
| 8.10 | Failed builds generate notifications to the platform team | **MEDIUM** | All |
| 8.11 | Pipeline artifacts (build logs, scan reports, test results) are retained for audit | **MEDIUM** | All |
| 8.12 | Pipeline secrets (vCenter credentials, SSH keys) are managed via CI/CD secret storage (not in repository) | **CRITICAL** | All |
| 8.13 | Template build environment is isolated (dedicated build network, limited outbound access) | **MEDIUM** | All |

### Example Correct Configuration

**GitHub Actions pipeline for golden image build**:

```yaml
name: Golden Image Build

on:
  schedule:
    - cron: '0 2 * * 3'    # Weekly Wednesday at 2 AM
  workflow_dispatch:
    inputs:
      os:
        description: 'OS to build'
        required: true
        type: choice
        options:
          - rhel9
          - ubuntu2204
          - windows2022
      reason:
        description: 'Reason for manual build'
        required: true
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  build:
    runs-on: [self-hosted, packer-builder]
    timeout-minutes: 120
    strategy:
      matrix:
        os: ${{ github.event_name == 'workflow_dispatch' && fromJSON(format('["{0}"]', inputs.os)) || fromJSON('["rhel9", "ubuntu2204", "windows2022"]') }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Packer
        uses: hashicorp/setup-packer@v3
        with:
          version: '1.11.0'

      - name: Packer Init
        run: packer init templates/${{ matrix.os }}/

      - name: Packer Validate
        run: packer validate templates/${{ matrix.os }}/

      - name: Packer Build
        run: packer build templates/${{ matrix.os }}/
        env:
          PKR_VAR_vcenter_server: ${{ secrets.VCENTER_SERVER }}
          PKR_VAR_vcenter_username: ${{ secrets.VCENTER_USERNAME }}
          PKR_VAR_vcenter_password: ${{ secrets.VCENTER_PASSWORD }}
          PKR_VAR_ssh_password: ${{ secrets.TEMPLATE_SSH_PASSWORD }}
          PKR_VAR_build_version: "1.${{ github.run_number }}"

      - name: Upload build log
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: packer-log-${{ matrix.os }}
          path: packer-build.log
          retention-days: 90

  test:
    needs: build
    runs-on: [self-hosted, packer-builder]
    strategy:
      matrix:
        os: ${{ github.event_name == 'workflow_dispatch' && fromJSON(format('["{0}"]', inputs.os)) || fromJSON('["rhel9", "ubuntu2204", "windows2022"]') }}
    steps:
      - uses: actions/checkout@v4

      - name: Deploy test VM from template
        run: |
          python3 scripts/deploy-test-vm.py \
            --template "${{ matrix.os }}-cis-v1.${{ github.run_number }}" \
            --customization-spec "${{ matrix.os }}-test" \
            --wait-for-tools 300

      - name: Run validation tests
        run: |
          python3 scripts/validate-template.py \
            --vm-name "test-${{ matrix.os }}-${{ github.run_number }}" \
            --tests boot,tools,network,customization,ssh-keys

      - name: Cleanup test VM
        if: always()
        run: |
          python3 scripts/destroy-test-vm.py \
            --vm-name "test-${{ matrix.os }}-${{ github.run_number }}"

  compliance-scan:
    needs: build
    runs-on: [self-hosted, packer-builder]
    strategy:
      matrix:
        os: ${{ github.event_name == 'workflow_dispatch' && fromJSON(format('["{0}"]', inputs.os)) || fromJSON('["rhel9", "ubuntu2204", "windows2022"]') }}
    steps:
      - uses: actions/checkout@v4

      - name: Deploy scan VM
        run: |
          python3 scripts/deploy-test-vm.py \
            --template "${{ matrix.os }}-cis-v1.${{ github.run_number }}" \
            --customization-spec "${{ matrix.os }}-test" \
            --wait-for-tools 300

      - name: Run CIS compliance scan
        run: |
          ssh scan-vm "sudo oscap xccdf eval \
            --profile xccdf_org.ssgproject.content_profile_cis \
            --results /tmp/oscap-results.xml \
            --report /tmp/oscap-report.html \
            /usr/share/xml/scap/ssg/content/ssg-${{ matrix.os }}-ds.xml"

      - name: Run vulnerability scan
        run: |
          trivy vm --severity HIGH,CRITICAL \
            --exit-code 1 \
            "vsphere://vcenter.example.com/DC-01/vm/test-${{ matrix.os }}"

      - name: Upload compliance report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-${{ matrix.os }}
          path: reports/
          retention-days: 365

      - name: Cleanup scan VM
        if: always()
        run: |
          python3 scripts/destroy-test-vm.py \
            --vm-name "test-${{ matrix.os }}-${{ github.run_number }}"

  promote:
    needs: [test, compliance-scan]
    runs-on: [self-hosted, packer-builder]
    strategy:
      matrix:
        os: ${{ github.event_name == 'workflow_dispatch' && fromJSON(format('["{0}"]', inputs.os)) || fromJSON('["rhel9", "ubuntu2204", "windows2022"]') }}
    steps:
      - name: Promote template to content library
        run: |
          python3 scripts/promote-template.py \
            --template "${{ matrix.os }}-cis-v1.${{ github.run_number }}" \
            --content-library "Production-Templates" \
            --deprecate-previous

      - name: Notify team
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"Golden image ${{ matrix.os }} v1.${{ github.run_number }} promoted to production.\"}"
```

### Common Mistakes

**Mistake**: Templates built on developer workstations with no CI/CD pipeline.

Why it is a problem: Manual builds are unreproducible, undocumented, and inconsistent. Different builders may install different packages, apply different hardening, or forget cleanup steps. There is no audit trail, no compliance validation, and no automated testing. If the builder's workstation is compromised, malware can be injected into the template.

**Mistake**: Pipeline builds templates but does not test them.

Why it is a problem: A template that builds successfully may still fail during deployment. Common issues only discovered through testing: cloud-init fails to run on first boot, guest tools are not functional, network customization does not apply, SELinux denials prevent services from starting, or the template exceeds a size limit for the deployment workflow.

**Mistake**: No vulnerability scanning of templates before promotion.

Why it is a problem: OS vendor patches may not cover all installed software. Third-party packages, agents, and libraries may have known CVEs. Without vulnerability scanning, templates are promoted to production with exploitable vulnerabilities that only become apparent after deployment (or after an incident).

---

## 9. Packer Best Practices

**Category**: Build Automation

Beyond the template-specific configuration, Packer itself has operational best practices that affect reliability, security, and maintainability of the image pipeline.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 9.1 | Packer plugins are pinned to specific versions in `required_plugins` blocks | **MEDIUM** | All |
| 9.2 | Packer builds use the `vsphere-iso` or `proxmox-iso` builder (not the deprecated `vmware-iso` for vSphere targets) | **LOW** | vSphere, Proxmox |
| 9.3 | HTTP server directory used for kickstart/preseed does not contain secrets | **HIGH** | All |
| 9.4 | Packer communicator timeout is set to a reasonable value (30-60 minutes) to handle slow installations | **LOW** | All |
| 9.5 | Packer error cleanup mode is set to retain artifacts on failure for debugging (`-on-error=ask` in development, `-on-error=cleanup` in CI) | **LOW** | All |
| 9.6 | Packer builds are run with `-timestamp-ui` for log correlation | **LOW** | All |
| 9.7 | Packer HCL2 files are formatted with `packer fmt` and validated with `packer validate` in CI | **LOW** | All |
| 9.8 | Packer provisioner ordering follows: update OS, install packages, harden, configure agents, cleanup (in that order) | **MEDIUM** | All |
| 9.9 | Shell provisioners use `set -euo pipefail` (or equivalent error handling) to fail on any error | **MEDIUM** | All |
| 9.10 | Ansible provisioners pin the Ansible version and collection versions for reproducibility | **MEDIUM** | All |
| 9.11 | Packer builds are tested in a non-production environment before promoting template definitions to production | **MEDIUM** | All |
| 9.12 | Packer build logs are captured and archived for audit and troubleshooting | **LOW** | All |
| 9.13 | `.pkrvars.hcl` files containing secrets are in `.gitignore` | **HIGH** | All |
| 9.14 | Packer data source integrations (Vault, AWS SSM) are used for dynamic secret retrieval | **MEDIUM** | All |

### Common Mistakes

**Mistake**: Shell provisioners without error handling.

```bash
# BAD - errors are silently ignored, template may be incomplete
#!/bin/bash
dnf update -y
dnf install -y open-vm-tools
systemctl enable vmtoolsd
# If dnf install fails, the script continues and the template
# is created without VMware Tools

# GOOD - fail on any error
#!/bin/bash
set -euo pipefail
dnf update -y
dnf install -y open-vm-tools
systemctl enable vmtoolsd
```

**Mistake**: Unpinned Packer plugin versions.

```hcl
# BAD - no version constraint, may pull breaking changes
packer {
  required_plugins {
    vsphere = {
      source = "github.com/hashicorp/vsphere"
    }
  }
}

# GOOD - pinned to specific version
packer {
  required_plugins {
    vsphere = {
      version = "~> 1.3.0"
      source  = "github.com/hashicorp/vsphere"
    }
  }
}
```

**Mistake**: Kickstart/preseed file served via Packer HTTP server containing a root password hash.

Why it is a problem: While the HTTP server is ephemeral and typically only accessible during the build, the kickstart file is stored in version control. The password hash can be brute-forced offline. Use a temporary password in the kickstart and change it during the provisioning phase, or use a separate mechanism for credential injection.

---

## 10. Multi-Platform Template Strategy

**Category**: Template Governance

Organizations running multiple hypervisor platforms (vSphere, Proxmox, KVM, cloud) need a unified template strategy. Templates should share hardening baselines, comply with the same security standards, and use consistent tooling, even if the output formats differ.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 10.1 | A single Packer source template can produce images for multiple target platforms (multi-builder) | **LOW** | All |
| 10.2 | Hardening playbooks/scripts are shared across platforms (not duplicated per builder) | **MEDIUM** | All |
| 10.3 | Compliance standards (CIS benchmark level, hardening version) are consistent across platforms | **HIGH** | All |
| 10.4 | Template naming conventions are consistent across platforms | **LOW** | All |
| 10.5 | Template lifecycle (build schedule, deprecation policy) is consistent across platforms | **MEDIUM** | All |
| 10.6 | Template inventory tracks images across all platforms with cross-reference | **MEDIUM** | All |

### Example Correct Configuration

**Packer multi-builder configuration**:

```hcl
# templates/rhel9/main.pkr.hcl
# Shared source configuration, multiple builders

source "vsphere-iso" "rhel9" {
  # vSphere-specific settings
  vcenter_server = var.vcenter_server
  # ... (as shown in Section 6)
}

source "proxmox-iso" "rhel9" {
  # Proxmox-specific settings
  proxmox_url = var.proxmox_url
  # ... (as shown in Section 6)
}

source "qemu" "rhel9" {
  # KVM/QEMU-specific settings
  output_directory = "output-rhel9"
  format           = "qcow2"
  disk_size        = "40G"
  accelerator      = "kvm"
  # ...
}

build {
  sources = [
    "source.vsphere-iso.rhel9",
    "source.proxmox-iso.rhel9",
    "source.qemu.rhel9"
  ]

  # Shared provisioning steps - identical hardening across platforms
  provisioner "shell" {
    inline = [
      "dnf update -y",
      "dnf install -y cloud-init"
    ]
  }

  provisioner "ansible" {
    playbook_file = "${path.root}/ansible/cis-hardening.yml"
  }

  provisioner "shell" {
    script = "${path.root}/scripts/cleanup.sh"
  }

  # Platform-specific post-processing
  post-processor "shell-local" {
    only   = ["qemu.rhel9"]
    inline = ["qemu-img convert -O qcow2 -c output-rhel9/rhel9 output-rhel9/rhel9-compressed.qcow2"]
  }
}
```

---

## Review Procedure Summary

When evaluating template and golden image management:

1. **Assess template creation**: Verify templates use minimal OS installations with CIS benchmark hardening, guest tools, and cloud-init/sysprep.
2. **Check credential cleanup**: Confirm SSH keys, authorized_keys, shell history, builder accounts, sudo entries, and machine-id are removed before templating.
3. **Validate customization**: Test that cloud-init and sysprep produce unique identifiers (hostname, SID, machine-id, SSH host keys) on every deployment.
4. **Review template lifecycle**: Verify templates are versioned, regularly rebuilt (not patched in-place), and deprecated with documented sunset dates.
5. **Evaluate automation**: Confirm Packer or equivalent automation is used, templates are version-controlled, and sensitive values are not in source code.
6. **Inspect the CI/CD pipeline**: Verify the pipeline includes build, test (deploy and validate), compliance scan, vulnerability scan, and promotion stages.
7. **Check pipeline security**: Confirm secrets are managed via CI/CD secret storage, build environments are isolated, and logs are retained for audit.
8. **Verify multi-platform consistency**: If multiple hypervisors are in use, confirm hardening baselines and lifecycle processes are consistent.
9. **Classify each finding** using the severity levels in each section's checkpoint table.

---

## Quick Reference: Severity Summary

| Severity | Template Management Findings |
|----------|------------------------------|
| CRITICAL | SSH authorized_keys with builder's public key in template; sudo NOPASSWD for builder account; API tokens or credentials left in template; plaintext passwords in sysprep/unattend.xml; pipeline secrets in version control; Ceph/vCenter credentials hardcoded in Packer files |
| HIGH | No CIS or hardening baseline applied; firewall disabled in template; SSH root login enabled; guest tools not installed; cloud-init not installed; SSH host keys not removed; builder accounts not removed; machine-id not cleaned; templates not rebuilt regularly (>6 months stale); no compliance scanning; no vulnerability scanning; templates created manually without automation; Packer templates not in version control; pipeline does not test deployed templates; SELinux/AppArmor disabled in template; kickstart directory contains secrets; customization spec domain credentials exposed |
| MEDIUM | Template not minimal (desktop packages installed); unnecessary services running; weak password policy; shell history not cleared; network config baked in; persistent udev rules present; template naming inconsistent; no version metadata; old templates not deprecated; build schedule misaligned with patch cycles; Packer variables not parameterized; provisioner ordering incorrect; shell provisioners without error handling; Ansible versions unpinned; hardening inconsistent across platforms; customization not tested; no pipeline failure notifications |
| LOW | Template disk not aligned to 4K sectors; hardware version not optimal; no template rollback capability; deprecated templates not clearly marked; Packer HCL2 formatting not enforced; build timeout not configured; build artifacts not cleaned up; Packer logs not archived; template annotations incomplete; cloud-init final_message not configured; multi-builder not used for multi-platform |
| INFO | Automated golden image pipeline with build/test/scan/promote stages; CIS benchmark scores tracked over time; template inventory with cross-platform tracking; Packer builds integrated with HashiCorp Vault for secret retrieval; compliance scan results archived for audit |

---

## References

- CIS Benchmarks for Red Hat Enterprise Linux 9, Ubuntu 22.04, Windows Server 2022
- HashiCorp Packer Documentation: vSphere, Proxmox, QEMU Builders
- HashiCorp Packer Best Practices Guide
- VMware vSphere 8 VM Customization Guide
- VMware Content Library Administration
- Microsoft Sysprep Documentation: Windows Server 2022
- cloud-init Documentation: DataSources, Modules, Best Practices
- OpenSCAP Project: Security Content Automation Protocol
- NIST SP 800-123: Guide to General Server Security
- NIST SP 800-70: National Checklist Program for IT Products
- DISA STIGs: Red Hat Enterprise Linux, Windows Server, VMware
- Proxmox VE Administration Guide: Cloud-Init, Templates
- Red Hat Virtualization: Template Best Practices
- OWASP Software Component Verification Standard (for image supply chain)
