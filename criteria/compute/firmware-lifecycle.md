# Firmware Lifecycle Management

Reference checklist for the `compute-reviewer` agent when evaluating firmware inventory, update processes, vulnerability management, and end-of-life planning for server hardware. This file complements the inline criteria in `review-compute.md` with detailed guidance on firmware version tracking, update procedures, CVE remediation, and hardware lifecycle management across Dell, HPE, and Supermicro platforms.

---

## 1. Firmware Inventory Management

**Category**: Firmware Inventory

A complete and accurate firmware inventory is the foundation of vulnerability management and lifecycle planning. Without knowing what firmware versions are running across the fleet, organizations cannot assess exposure to CVEs, plan update campaigns, or verify compliance with vendor recommendations.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A centralized firmware inventory exists covering all server components | HIGH | Without inventory, firmware vulnerabilities cannot be assessed fleet-wide |
| BMC/iDRAC/iLO firmware version is tracked for every server | HIGH | BMC firmware is the most security-critical component; it has OS-independent access |
| BIOS/UEFI firmware version is tracked | HIGH | BIOS vulnerabilities (e.g., LogoFAIL, PixieFail) enable pre-boot attacks |
| NIC firmware and driver versions are tracked (Intel, Broadcom, Mellanox) | MEDIUM | NIC firmware vulnerabilities can enable remote code execution or denial of service |
| HBA/FC adapter firmware is tracked (QLogic, Emulex/Broadcom) | MEDIUM | HBA firmware issues can cause storage connectivity failures or data corruption |
| RAID controller firmware is tracked (PERC, Smart Array, MegaRAID) | HIGH | RAID controller firmware bugs can cause data loss, array degradation, or rebuild failures |
| SSD/NVMe firmware is tracked | HIGH | SSD firmware bugs can cause data loss (e.g., HPE SAS SSD 32,768-hour bug, Intel SSD DC firmware issues) |
| HDD firmware is tracked for spinning disks | MEDIUM | Less frequently updated but firmware bugs can cause silent data corruption |
| GPU firmware is tracked (if applicable) | LOW | GPU firmware updates are less frequent but relevant for AI/ML workloads |
| CPLD (Complex Programmable Logic Device) firmware is tracked | LOW | CPLD firmware manages low-level platform functions; rarely updated but critical when needed |
| PSU (Power Supply Unit) firmware is tracked | LOW | PSU firmware bugs can cause unexpected shutdowns or efficiency degradation |
| Backplane/midplane firmware is tracked | LOW | Backplane firmware issues can cause intermittent disk connectivity |
| Inventory is automated (not manual spreadsheets) | MEDIUM | Manual inventory is always incomplete and outdated; automation ensures accuracy |
| Inventory is updated on a defined schedule (at least quarterly) | MEDIUM | Stale inventory misses firmware changes from maintenance activities |
| Inventory includes serial numbers, service tags, and warranty status | MEDIUM | Required for vendor support cases, RMA, and lifecycle planning |

### Common Mistakes

**Mistake**: Tracking only BMC and BIOS firmware, ignoring component firmware.

Why it is a problem: The 2019 HPE SAS SSD firmware bug (HPD8 firmware) caused drives to fail after exactly 32,768 hours of operation. Organizations that did not track SSD firmware versions could not proactively identify affected drives before they failed, resulting in data loss across arrays.

**Mistake**: Maintaining firmware inventory in a spreadsheet updated manually after maintenance windows.

Why it is a problem: Manual inventory is always incomplete. Servers added outside the change process, emergency firmware updates, or firmware downgrades during troubleshooting create discrepancies. By the time a CVE advisory arrives, the spreadsheet is already wrong.

### Fix Patterns

**Automated firmware inventory collection via Redfish API**:

```bash
#!/bin/bash
# collect-firmware-inventory.sh
# Queries Redfish API to collect firmware inventory from all servers

SERVERS_FILE="/etc/firmware-inventory/servers.txt"
OUTPUT_DIR="/var/lib/firmware-inventory"
DATE=$(date +%Y%m%d)

mkdir -p "${OUTPUT_DIR}/${DATE}"

while IFS=, read -r hostname bmc_ip vendor; do
  echo "Collecting firmware from ${hostname} (${bmc_ip})..."

  # Redfish firmware inventory endpoint (standard across vendors)
  curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
    "https://${bmc_ip}/redfish/v1/UpdateService/FirmwareInventory" \
    | jq '.Members[] | {Name, Version, Id, Updateable}' \
    > "${OUTPUT_DIR}/${DATE}/${hostname}_firmware.json"

  # Also collect system model and serial
  curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
    "https://${bmc_ip}/redfish/v1/Systems/System.Embedded.1" \
    | jq '{Model, Manufacturer, SerialNumber, BiosVersion, SKU}' \
    > "${OUTPUT_DIR}/${DATE}/${hostname}_system.json"

done < "${SERVERS_FILE}"

# Generate fleet summary report
echo "Hostname,Component,CurrentVersion" > "${OUTPUT_DIR}/${DATE}/fleet_summary.csv"
for f in "${OUTPUT_DIR}/${DATE}"/*_firmware.json; do
  hostname=$(basename "$f" _firmware.json)
  jq -r --arg h "$hostname" '"\($h),\(.Name),\(.Version)"' "$f" \
    >> "${OUTPUT_DIR}/${DATE}/fleet_summary.csv"
done
```

**Dell iDRAC firmware inventory via racadm**:

```bash
# List all firmware versions on a Dell server
racadm swinventory

# Output includes:
# BIOS: 2.19.1
# iDRAC: 7.00.00.171
# PERC H755 Front: 52.28.0-4708
# NIC BCM57414: 22.31.8.12
# Disk SSD PM9A3: GDC5602Q
# CPLD: 1.1.8
# PSU: 00.2D.4A

# Export to file for comparison
racadm swinventory > /var/lib/firmware-inventory/$(hostname)_$(date +%Y%m%d).txt
```

**HPE iLO firmware inventory via ilorest**:

```bash
# List all firmware versions on an HPE server
ilorest get --json --select SoftwareInventory. | jq '.[] | {Name, Version}'

# Or via iLO web API
curl -sk -u admin:password \
  "https://ilo-server01/redfish/v1/UpdateService/FirmwareInventory/" \
  | jq '.Members[]."@odata.id"' | while read -r uri; do
    curl -sk -u admin:password "https://ilo-server01${uri}" \
      | jq '{Name, Version, Id}'
  done
```

**Ansible firmware inventory collection**:

```yaml
# roles/firmware-inventory/tasks/main.yml
---
- name: Collect firmware inventory via Redfish
  community.general.redfish_info:
    baseuri: "{{ bmc_address }}"
    username: "{{ bmc_user }}"
    password: "{{ bmc_password }}"
    category: Update
    command: GetFirmwareInventory
  register: firmware_inventory
  delegate_to: localhost

- name: Store firmware inventory
  ansible.builtin.copy:
    content: "{{ firmware_inventory | to_nice_json }}"
    dest: "/var/lib/firmware-inventory/{{ inventory_hostname }}_firmware.json"
  delegate_to: localhost

- name: Alert on outdated firmware
  ansible.builtin.debug:
    msg: "WARNING: {{ item.Name }} is at version {{ item.Version }}, minimum is {{ min_versions[item.Name] }}"
  loop: "{{ firmware_inventory.redfish_facts.firmware.entries }}"
  when:
    - item.Name in min_versions
    - item.Version is version(min_versions[item.Name], '<')
```

---

## 2. Version Tracking and Vendor Recommendations

**Category**: Firmware Currency

Firmware currency -- how current firmware versions are relative to vendor recommendations -- directly impacts security posture and system stability. Vendors publish firmware updates to fix bugs, patch vulnerabilities, and improve performance. Running outdated firmware means running with known issues.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| BMC firmware is within one major version of vendor's current recommended release | HIGH | BMC firmware trails create exposure to known RCE vulnerabilities |
| BIOS firmware is within vendor's current recommended release for the server model | HIGH | BIOS vulnerabilities enable pre-boot attacks that survive OS reinstallation |
| RAID controller firmware matches vendor's minimum recommended version | HIGH | Below-minimum RAID firmware risks data integrity issues and rebuild failures |
| NIC firmware matches vendor's recommended version for the driver in use | MEDIUM | NIC firmware/driver version mismatches cause connectivity issues and performance degradation |
| SSD/NVMe firmware is at or above vendor's minimum safe version | HIGH | SSD firmware bugs can cause sudden data loss (known issues documented in vendor advisories) |
| Firmware versions are compared against vendor support matrices | MEDIUM | Some firmware combinations are explicitly unsupported and may cause undefined behavior |
| Vendor firmware advisories are monitored (Dell Security Advisories, HPE Customer Bulletins) | MEDIUM | Without monitoring, critical firmware advisories are missed |
| Firmware update packages are available and tested before the maintenance window | MEDIUM | Discovering missing packages during a maintenance window wastes the window |
| Firmware is not ahead of vendor's recommended version (no running beta/pre-release) | LOW | Pre-release firmware may introduce new bugs; vendor support may not cover issues |
| Component firmware compatibility matrix is consulted before updates | MEDIUM | Some firmware updates require specific BIOS or BMC versions first |

### Common Mistakes

**Mistake**: Running BMC firmware that is more than two years behind vendor's current release.

Why it is a problem: BMC vulnerabilities are discovered regularly. For example:
- CVE-2019-6260 (USBAnywhere): Remote BMC access via virtual USB
- CVE-2023-34329: HPE iLO authentication bypass
- CVE-2022-40242: AMI MegaRAC BMC remote code execution
- CVE-2024-54085: AMI MegaRAC SPx BMC remote code execution

Organizations running BMC firmware more than 6 months behind vendor recommendations are likely vulnerable to at least one known CVE.

**Mistake**: Updating NIC firmware without matching the NIC driver version.

Why it is a problem: NIC firmware and driver versions must be compatible. Updating firmware without updating the driver (or vice versa) can cause link flaps, performance degradation, or complete loss of network connectivity.

```bash
# Check current NIC firmware and driver versions
ethtool -i eth0
# driver: i40e
# version: 2.22.18
# firmware-version: 9.20 0x8000d95e 22.0.9

# Dell: Verify compatibility via DSU catalog
dell-system-update --check-inventory

# HPE: Verify via SPP compatibility matrix
# Download Smart Update Manager (SUM) and run inventory
hpsum inventory --report /var/log/hpsum/inventory_$(date +%Y%m%d).html
```

**Mistake**: Not consulting the firmware dependency/ordering matrix before updating.

```
# BAD - updating components in wrong order can brick the BMC
1. Update NIC firmware
2. Update BIOS
3. Update iDRAC  <-- iDRAC should often be updated FIRST

# GOOD - Dell recommended update order (general guideline)
1. iDRAC firmware (BMC first -- it manages the update process)
2. BIOS
3. RAID controller
4. NIC firmware
5. Drive firmware
6. CPLD (if needed -- often requires specific iDRAC version)

# HPE recommended update order
1. iLO firmware
2. System ROM (BIOS)
3. Smart Array controller
4. NIC firmware
5. Drive firmware
6. Innovation Engine (IE) firmware
```

### Fix Patterns

**Dell DSU (Dell System Update) for automated updates**:

```bash
# Install Dell System Update
dnf install -y dell-system-update

# Check available updates against Dell catalog
dell-system-update --check-inventory

# Apply all recommended updates (requires reboot for BIOS/iDRAC)
dell-system-update --apply-upgrades --auto-reboot

# Apply updates non-interactively (for automation)
dell-system-update --apply-upgrades --reboot-type=1 --non-interactive

# Check specific component
dell-system-update --check-inventory --component-type=BIOS
```

**HPE SPP (Service Pack for ProLiant) updates**:

```bash
# Mount SPP ISO
mount -o loop /path/to/SPP.iso /mnt/spp

# Run Smart Update Manager in CLI mode
/mnt/spp/launch_sum.sh --s --use_latest \
  --reboot_message "Firmware update - planned maintenance" \
  --reboot_delay 60

# Or use iLO for individual component updates
ilorest flashfirmware /path/to/firmware.bin --tpmover
```

**Redfish API firmware update (vendor-neutral)**:

```bash
# Step 1: Upload firmware image
TASK=$(curl -sk -X POST \
  -u "${REDFISH_USER}:${REDFISH_PASS}" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @firmware_update.bin \
  "https://${BMC_IP}/redfish/v1/UpdateService/update" \
  | jq -r '.TaskId // .Id')

# Step 2: Monitor update task
while true; do
  STATUS=$(curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
    "https://${BMC_IP}/redfish/v1/TaskService/Tasks/${TASK}" \
    | jq -r '.TaskState')
  echo "Update status: ${STATUS}"
  [ "${STATUS}" = "Completed" ] && break
  [ "${STATUS}" = "Exception" ] && echo "FAILED" && exit 1
  sleep 30
done

# Step 3: Verify new version
curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
  "https://${BMC_IP}/redfish/v1/UpdateService/FirmwareInventory" \
  | jq '.Members[] | select(.Name | test("BMC|iDRAC|iLO")) | {Name, Version}'
```

---

## 3. Update Process and Testing

**Category**: Firmware Update Process

Firmware updates carry inherent risk -- a failed BMC update can brick a server, a RAID controller update during I/O can corrupt data, and an untested BIOS update can change boot behavior. A rigorous update process with testing, rollback plans, and post-update validation is essential.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Firmware updates are tested on non-production hardware before fleet rollout | HIGH | Untested firmware updates on production hardware risk fleet-wide failures |
| A rollback procedure is documented and tested for each firmware component | HIGH | Without rollback capability, a bad firmware update requires RMA or advanced recovery |
| Firmware update procedures are documented in a runbook | MEDIUM | Undocumented procedures are inconsistently executed and cannot be delegated |
| Pre-update health checks are performed (RAID status, memory errors, BMC health) | MEDIUM | Updating firmware on already-degraded hardware compounds the risk |
| Firmware updates are performed during scheduled maintenance windows | MEDIUM | Unscheduled firmware updates during business hours risk unexpected outages |
| Host-level services are gracefully drained before firmware updates requiring reboot | HIGH | Rebooting without draining causes service interruptions and potential data loss |
| Post-update validation confirms new firmware version and hardware health | MEDIUM | Without validation, a failed or partial update goes undetected |
| Firmware update logs are retained for audit trail | LOW | Required for change management compliance (ITIL, SOC 2) |
| Firmware updates that require reboot are batched to minimize maintenance windows | LOW | Separate reboots for BIOS and iDRAC when they could be combined wastes maintenance time |
| Canary/rolling update strategy is used for fleet-wide firmware campaigns | HIGH | Updating all servers simultaneously risks fleet-wide failure if the firmware has a defect |
| Emergency firmware updates have a defined SLA and expedited process | MEDIUM | Critical CVEs in BMC firmware cannot wait for the next quarterly maintenance window |
| Firmware update packages are verified (checksum/signature) before application | MEDIUM | Corrupted or tampered firmware packages can brick hardware or introduce backdoors |

### Common Mistakes

**Mistake**: Applying firmware updates to the entire fleet simultaneously.

Why it is a problem: If a firmware update has a defect (e.g., specific hardware revision incompatibility, race condition during update), updating all servers at once means all servers are affected. A canary approach -- updating 1-2 servers, validating for 24-48 hours, then updating in rolling batches -- limits blast radius.

```
# BAD - update all 200 servers in one maintenance window
for server in $(cat all_servers.txt); do
  ssh $server "dell-system-update --apply-upgrades --auto-reboot"
done

# GOOD - canary then rolling batches
# Phase 1: Canary (1-2 servers, non-critical workloads)
# Phase 2: Wait 48 hours, validate
# Phase 3: Batch 1 (10% of fleet)
# Phase 4: Wait 24 hours, validate
# Phase 5: Batch 2 (25% of fleet)
# Phase 6: Wait 24 hours, validate
# Phase 7: Remaining fleet
```

**Mistake**: No rollback plan for firmware updates.

Why it is a problem: BMC firmware can usually be rolled back to the previous version (Dell iDRAC stores the previous image). BIOS can often be rolled back. But RAID controller and drive firmware rollbacks are vendor-specific and not always supported. If a RAID controller firmware update causes issues, the only option may be to wait for a vendor patch.

```bash
# Dell iDRAC: Rollback to previous firmware
racadm rollback iDRAC

# Dell BIOS: Rollback to previous version
racadm rollback BIOS

# HPE iLO: Rollback to previous firmware
ilorest set FlashOnReboot=True --select UpdateService. --commit
# Then select the backup image

# Supermicro: BMC recovery via physical jumper (last resort)
# Requires physical access -- not ideal for remote data centers
```

**Mistake**: Skipping pre-update health checks.

```bash
# Pre-update health check script
#!/bin/bash
HOSTNAME=$(hostname)
ERRORS=0

# Check RAID status
echo "=== RAID Status ==="
if command -v omreport &>/dev/null; then
  # Dell OMSA
  omreport storage vdisk | grep -i status
  [ $? -ne 0 ] && ERRORS=$((ERRORS+1))
elif command -v ssacli &>/dev/null; then
  # HPE Smart Array
  ssacli ctrl all show status
  ssacli ctrl slot=0 ld all show status
fi

# Check for memory errors (correctable ECC errors indicate failing DIMMs)
echo "=== Memory Errors ==="
edac-util -s 2>/dev/null || echo "edac-utils not installed"
grep -i "Hardware Error" /var/log/mcelog 2>/dev/null

# Check BMC health
echo "=== BMC Health ==="
ipmitool sel elist last 20

# Check disk health (SMART)
echo "=== Disk Health ==="
for disk in /dev/sd[a-z]; do
  smartctl -H "$disk" 2>/dev/null | grep -i "result\|status"
done

# Check current firmware versions (for rollback reference)
echo "=== Current Firmware Versions ==="
racadm swinventory 2>/dev/null || ilorest get --select SoftwareInventory. 2>/dev/null

echo "=== Pre-update check complete. Errors: ${ERRORS} ==="
[ ${ERRORS} -gt 0 ] && echo "ABORT: Resolve errors before firmware update" && exit 1
```

### Fix Patterns

**Post-update validation script**:

```bash
#!/bin/bash
# post-firmware-validation.sh
# Run after firmware update and reboot to verify success

EXPECTED_VERSIONS_FILE="/etc/firmware-baseline/expected_versions.json"
ERRORS=0

echo "=== Post-Firmware-Update Validation ==="
echo "Date: $(date)"
echo "Hostname: $(hostname)"

# 1. Verify firmware versions match expected
echo "--- Firmware Version Check ---"
# Compare current vs expected (vendor-specific collection)
if command -v racadm &>/dev/null; then
  CURRENT_IDRAC=$(racadm getversion -f idrac)
  CURRENT_BIOS=$(racadm getversion -f bios)
  echo "iDRAC: ${CURRENT_IDRAC}"
  echo "BIOS: ${CURRENT_BIOS}"
fi

# 2. Verify hardware health post-update
echo "--- Hardware Health ---"
ipmitool sdr type "Power Supply" | grep -v "ok"
ipmitool sdr type "Temperature" | grep -v "ok"
ipmitool sdr type "Fan" | grep -v "ok"

# 3. Verify RAID status
echo "--- RAID Status ---"
if command -v omreport &>/dev/null; then
  omreport storage vdisk | grep -E "Status|State"
fi

# 4. Verify network interfaces are up
echo "--- Network Interfaces ---"
for iface in $(ip -br link show | awk '{print $1}'); do
  STATE=$(ip -br link show "$iface" | awk '{print $2}')
  [ "$STATE" != "UP" ] && [ "$iface" != "lo" ] && echo "WARNING: $iface is $STATE"
done

# 5. Check for new SEL entries (errors during update)
echo "--- System Event Log (post-update) ---"
ipmitool sel elist last 10

# 6. Verify Secure Boot status
echo "--- Secure Boot ---"
mokutil --sb-state 2>/dev/null || echo "mokutil not available"

# 7. Verify kernel loaded correctly (no panic/oops during boot)
echo "--- Kernel Boot ---"
dmesg | grep -iE "panic|oops|error|fail" | head -20

echo "=== Validation complete. Errors: ${ERRORS} ==="
```

---

## 4. CVE and Vulnerability Management

**Category**: Firmware Vulnerability Management

Server firmware is subject to the same vulnerability disclosure and patching lifecycle as software. However, firmware CVEs often receive less attention than OS or application CVEs, despite being more dangerous -- a firmware vulnerability can persist across OS reinstallation and provide hardware-level persistence to an attacker.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Firmware CVE monitoring is active (vendor security advisories, NIST NVD) | HIGH | Without monitoring, critical firmware CVEs go unpatched |
| Spectre/Meltdown/MDS mitigations are applied (CPU microcode + kernel patches) | HIGH | Speculative execution vulnerabilities affect all modern CPUs; mitigations require both microcode and kernel updates |
| Known critical BMC CVEs are patched within 30 days of vendor fix availability | HIGH | BMC RCE vulnerabilities are actively exploited; 30-day SLA aligns with industry best practice |
| Known critical BIOS CVEs are patched within 30 days | HIGH | BIOS vulnerabilities enable persistent pre-boot attacks |
| Speculative execution vulnerability status is tracked (`/sys/devices/system/cpu/vulnerabilities/`) | MEDIUM | Linux exposes per-vulnerability mitigation status; unmitigated vulnerabilities should be documented |
| Hardware end-of-life servers that no longer receive firmware updates are identified | HIGH | Servers without vendor firmware support cannot be patched and represent permanent risk |
| Firmware vulnerability scanning is part of regular security assessments | MEDIUM | Standard vulnerability scanners (Nessus, Qualys) can check firmware versions if properly configured |
| CVE remediation for firmware follows the same SLA as OS/application CVEs | MEDIUM | Treating firmware CVEs with lower priority than software CVEs is a common gap |
| Side-channel attack mitigations (Spectre v1/v2, Meltdown, MDS, L1TF, TAA, MMIO, Downfall, Inception, Zenbleed) are tracked | MEDIUM | New speculative execution variants are discovered regularly; each requires specific microcode + kernel + compiler mitigations |
| Microcode updates are applied at boot via OS package (not just BIOS update) | MEDIUM | OS-loaded microcode updates can be deployed faster than BIOS updates and do not require reboot to flash BIOS |

### Common Mistakes

**Mistake**: Relying solely on BIOS updates for CPU microcode.

Why it is a problem: BIOS updates are slow to roll out (require maintenance window, vendor packaging). Linux can load microcode at early boot from an OS package (`intel-microcode` or `amd64-microcode`), which can be deployed as a regular OS patch. Both methods should be used: OS-level microcode for rapid deployment, BIOS-level microcode for completeness.

```bash
# Check current microcode revision
cat /proc/cpuinfo | grep -m1 microcode
# microcode	: 0x2b000590

# Install microcode package (Debian/Ubuntu)
apt install intel-microcode  # or amd64-microcode

# Install microcode package (RHEL/CentOS)
dnf install microcode_ctl

# Verify microcode is loaded early at boot
dmesg | grep microcode
# [    0.000000] microcode: microcode updated early to revision 0x2b000590
```

**Mistake**: Not tracking speculative execution vulnerability mitigation status.

```bash
# Check all speculative execution vulnerability mitigations
for vuln in /sys/devices/system/cpu/vulnerabilities/*; do
  echo "$(basename $vuln): $(cat $vuln)"
done

# Example output showing unmitigated vulnerability:
# itlb_multihit: KVM: Mitigation: VMX disabled
# l1tf: Mitigation: PTE Inversion; VMX: conditional cache flushes, SMT vulnerable
# mds: Mitigation: Clear CPU buffers; SMT vulnerable
# meltdown: Mitigation: PTI
# mmio_stale_data: Mitigation: Clear CPU buffers; SMT vulnerable
# retbleed: Mitigation: Enhanced IBRS
# spec_store_bypass: Mitigation: Speculative Store Bypass disabled via prctl
# spectre_v1: Mitigation: usercopy/swapgs barriers and __user pointer sanitization
# spectre_v2: Mitigation: Enhanced IBRS; IBPB: conditional; RSB filling; PBRSB-eIBRS: SW sequence
# srbds: Mitigation: Microcode
# tsx_async_abort: Mitigation: Clear CPU buffers; SMT vulnerable

# "Vulnerable" status for any entry requires investigation
grep -l "Vulnerable" /sys/devices/system/cpu/vulnerabilities/* 2>/dev/null
```

**Mistake**: Not tracking known SSD firmware bugs with data loss potential.

```
# Known critical SSD firmware issues (examples):
# - HPE SAS SSD HPD8 firmware: drive failure at 32,768 hours
# - Intel SSD DC S3520/S3610: firmware MCE040 data loss on power loss
# - Samsung PM863a: firmware GXT5404Q linked to unrecoverable errors
# - Micron 5200 ECO: firmware D1MU020 linked to sudden power loss vulnerability

# Check SSD firmware versions
smartctl -i /dev/sda | grep -i firmware
# Firmware Version:    GDC5602Q

# Compare against vendor advisory minimum firmware
# Dell: Check Dell Security Advisories (DSA) and Product Alerts
# HPE: Check HPE Customer Bulletins
# Direct from drive manufacturer (Samsung, Intel/Solidigm, Micron, WD/SanDisk)
```

### Fix Patterns

**Automated CVE checking against firmware inventory**:

```python
#!/usr/bin/env python3
"""
firmware_cve_check.py
Cross-references firmware inventory against known CVE database.
"""

import json
import sys
from pathlib import Path

# Known critical firmware CVEs (maintain this list from vendor advisories)
KNOWN_CVES = {
    "iDRAC": [
        {
            "cve": "CVE-2024-22453",
            "severity": "CRITICAL",
            "affected_below": "7.00.00.171",
            "description": "iDRAC9 stack-based buffer overflow",
            "advisory": "DSA-2024-072",
        },
        {
            "cve": "CVE-2023-44277",
            "severity": "HIGH",
            "affected_below": "6.10.80.00",
            "description": "iDRAC9 OS command injection",
            "advisory": "DSA-2023-422",
        },
    ],
    "iLO 5": [
        {
            "cve": "CVE-2023-34329",
            "severity": "CRITICAL",
            "affected_below": "2.72",
            "description": "iLO 5 authentication bypass",
            "advisory": "HPESBHF04466",
        },
    ],
    "MegaRAC BMC": [
        {
            "cve": "CVE-2024-54085",
            "severity": "CRITICAL",
            "affected_below": "varies",
            "description": "AMI MegaRAC SPx remote code execution",
            "advisory": "AMI-SA-2024004",
        },
    ],
}


def check_firmware(inventory_file: str) -> list:
    """Check firmware inventory against known CVEs."""
    findings = []
    with open(inventory_file) as f:
        inventory = json.load(f)

    for component in inventory:
        name = component.get("Name", "")
        version = component.get("Version", "")

        for cve_component, cves in KNOWN_CVES.items():
            if cve_component.lower() in name.lower():
                for cve in cves:
                    if version < cve["affected_below"]:
                        findings.append({
                            "hostname": inventory_file,
                            "component": name,
                            "current_version": version,
                            "cve": cve["cve"],
                            "severity": cve["severity"],
                            "fix_version": cve["affected_below"],
                            "advisory": cve["advisory"],
                        })
    return findings


if __name__ == "__main__":
    for inv_file in Path(sys.argv[1]).glob("*_firmware.json"):
        findings = check_firmware(str(inv_file))
        for f in findings:
            print(f"[{f['severity']}] {f['hostname']}: {f['component']} "
                  f"v{f['current_version']} vulnerable to {f['cve']} "
                  f"(fix: >= {f['fix_version']}, advisory: {f['advisory']})")
```

**Emergency patching SLA matrix**:

| CVE Severity (CVSS) | Firmware Type | Patching SLA | Approval Process |
|---------------------|---------------|--------------|-----------------|
| CRITICAL (9.0-10.0) | BMC (remote exploit, no auth) | 72 hours | Emergency change, CAB notification |
| CRITICAL (9.0-10.0) | BIOS/CPU microcode | 7 days | Emergency change, CAB approval |
| HIGH (7.0-8.9) | Any component | 30 days | Standard change window |
| MEDIUM (4.0-6.9) | Any component | 90 days | Next scheduled maintenance |
| LOW (0.1-3.9) | Any component | Next firmware campaign | Bundled with other updates |

---

## 5. Maintenance Window Coordination

**Category**: Change Management

Firmware updates often require server reboots, which impact running workloads. Proper coordination with application owners, cluster managers, and monitoring systems ensures that firmware maintenance does not cause unplanned service disruptions.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Firmware updates follow the organization's change management process | MEDIUM | Unauthorized changes bypass review and create audit gaps |
| Application teams are notified before firmware maintenance that requires reboot | HIGH | Unexpected reboots cause application outages and data loss |
| Workloads are gracefully drained before reboot (Kubernetes cordon/drain, VM live migration, service disable) | HIGH | Ungraceful shutdown causes in-flight transactions to fail and may corrupt data |
| Monitoring systems are informed of maintenance windows (alerting suppression) | MEDIUM | Firmware maintenance generates false alerts that desensitize on-call teams |
| Cluster-aware maintenance considers quorum requirements (e.g., no more than one node at a time in a 3-node cluster) | HIGH | Rebooting too many cluster members simultaneously breaks quorum and causes outage |
| Firmware update completion is verified before returning server to production | MEDIUM | Returning an improperly updated server causes degraded service |
| Maintenance window duration is estimated based on component types being updated | LOW | BIOS updates with reboot take 5-15 minutes; multiple component updates take 30-60 minutes |
| Backout/rollback procedure is defined before the maintenance window begins | HIGH | Discovering rollback is needed without a procedure extends the outage |

### Common Mistakes

**Mistake**: Rebooting servers for firmware updates without draining workloads.

```bash
# BAD - immediate reboot with running workloads
dell-system-update --apply-upgrades --auto-reboot

# GOOD - drain workloads first
# For Kubernetes nodes:
kubectl cordon ${NODE}
kubectl drain ${NODE} --ignore-daemonsets --delete-emptydir-data --timeout=300s
# Apply firmware updates
dell-system-update --apply-upgrades
# Reboot
systemctl reboot
# After reboot and validation:
kubectl uncordon ${NODE}

# For VMware ESXi hosts:
# Enter maintenance mode (triggers vMotion of all VMs)
esxcli system maintenanceMode set --enable true
# Apply firmware updates (HPE OneView, Dell OME, or manual)
# Reboot
reboot
# Exit maintenance mode
esxcli system maintenanceMode set --enable false
```

**Mistake**: Updating multiple nodes in a cluster simultaneously.

```bash
# BAD - parallel update of all cluster nodes
for node in node1 node2 node3; do
  ssh $node "dell-system-update --apply-upgrades --auto-reboot" &
done
wait

# GOOD - serial update with validation between nodes
for node in node1 node2 node3; do
  echo "=== Updating ${node} ==="
  # Drain
  kubectl cordon ${node}
  kubectl drain ${node} --ignore-daemonsets --delete-emptydir-data --timeout=300s

  # Update and reboot
  ssh ${node} "dell-system-update --apply-upgrades"
  ssh ${node} "systemctl reboot"

  # Wait for node to come back
  echo "Waiting for ${node} to reboot..."
  sleep 60
  until ssh ${node} "uptime" 2>/dev/null; do sleep 10; done

  # Validate
  ssh ${node} "/usr/local/bin/post-firmware-validation.sh"

  # Return to service
  kubectl uncordon ${node}

  # Wait for workloads to stabilize
  echo "Waiting 5 minutes for workload stabilization..."
  sleep 300
done
```

---

## 6. End-of-Life Planning

**Category**: Hardware Lifecycle

Servers have a finite lifecycle. When hardware reaches end-of-life (EOL) or end-of-service-life (EOSL), it no longer receives firmware updates, vendor support, or replacement parts. Running EOL hardware in production creates permanent security and reliability gaps.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Hardware warranty status is tracked for all servers | MEDIUM | Expired warranty means no vendor support for hardware failures; replacement parts may be unavailable |
| Vendor end-of-life dates are tracked for each server model | HIGH | EOL dates are published years in advance; failure to plan forces emergency procurement |
| Servers past vendor EOSL that no longer receive firmware updates are identified | HIGH | No firmware updates means no CVE remediation; these servers are permanently vulnerable |
| Hardware refresh cycle is defined (typically 4-5 years for servers) | MEDIUM | Without a refresh cycle, hardware ages until failure rather than planned replacement |
| Budget for hardware refresh is planned 12+ months in advance | MEDIUM | Hardware procurement lead times can be 3-6 months; budget planning must precede procurement |
| Decommissioning procedure includes secure data wiping | CRITICAL | Decommissioned drives and servers with data remnants are a data breach risk |
| Secure wipe uses vendor-approved methods (cryptographic erase, NIST 800-88 compliant) | CRITICAL | Simple format or single-pass overwrite is insufficient for regulatory compliance |
| Decommissioning includes BMC credential clearing | HIGH | BMC credentials on decommissioned hardware accessible to third parties is a credential leak |
| Asset disposition chain of custody is documented | MEDIUM | Required for compliance; proves data was properly destroyed |
| Extended warranty or third-party support is evaluated for servers kept beyond standard lifecycle | LOW | Third-party support (Park Place, CXtec) can extend usable life when vendor support ends |
| Capacity and performance of aging hardware is monitored for degradation | MEDIUM | Aging components (SSDs with high wear, fans, PSUs) degrade before complete failure |
| Spare parts inventory is maintained for production-critical servers outside warranty | MEDIUM | Failed components in out-of-warranty servers require spares on hand for timely repair |

### Common Mistakes

**Mistake**: Decommissioning servers without wiping data.

Why it is a problem: Drives removed from decommissioned servers and sent to e-waste or resold contain recoverable data. This applies to HDDs, SSDs (even "deleted" SSD data persists in overprovisioned NAND), and even NVDIMM/persistent memory modules. Multiple studies have found PII, financial data, and credentials on second-hand drives.

```bash
# GOOD - NIST 800-88 compliant cryptographic erase for SSD
# Method 1: ATA Secure Erase (for SATA SSDs)
hdparm --user-master u --security-set-pass password /dev/sda
hdparm --user-master u --security-erase-enhanced password /dev/sda

# Method 2: NVMe Format with cryptographic erase
nvme format /dev/nvme0n1 --ses=1  # Cryptographic erase (User Data Erase)
nvme format /dev/nvme0n1 --ses=2  # Cryptographic erase (all namespaces)

# Method 3: RAID controller secure erase (Dell PERC)
omconfig storage pdisk action=cryptographicerase \
  controller=0 pdisk=0:1:0

# Method 4: Vendor tools
# Dell: racadm storage secureerase:<FQDD>
# HPE: ssacli ctrl slot=0 pd 1I:1:1 modify erase erasepattern=zero

# Verify erasure
smartctl -a /dev/sda | grep -i "erase"
nvme id-ns /dev/nvme0n1 | grep -i "format"
```

**Mistake**: Not clearing BMC credentials before decommissioning.

```bash
# Dell iDRAC: Reset to factory defaults before decommissioning
racadm racresetcfg

# HPE iLO: Reset to factory defaults
ilorest set ResetBiosToDefaultsOnEachBoot=Enabled --select Bios. --commit
ilorest iloresetfactory

# Supermicro IPMI: Reset BMC to factory defaults
ipmitool -I lanplus -H ${BMC_IP} -U admin -P password mc reset cold
ipmitool -I lanplus -H ${BMC_IP} -U admin -P password raw 0x30 0x41
```

**Mistake**: No tracking of hardware approaching end-of-life.

```
# Hardware lifecycle tracking should include:
# - Purchase date
# - Warranty expiration date
# - Vendor end-of-sale date
# - Vendor end-of-support date (last firmware update date)
# - Planned refresh date
# - Current utilization (inform refresh priority)

# Example lifecycle timeline (Dell PowerEdge R650):
# Purchase:          2022-01-15
# Warranty expires:  2025-01-15 (3-year ProSupport)
# End of sale:       ~2025 (estimated)
# End of support:    ~2028 (estimated, 3 years after end-of-sale)
# Planned refresh:   2026-Q3 (4.5-year lifecycle)
```

### Fix Patterns

**Hardware lifecycle tracking in CMDB/inventory**:

```yaml
# hardware-lifecycle.yml - Track per server
servers:
  - hostname: db-prod-01
    model: Dell PowerEdge R750
    service_tag: ABC1234
    purchase_date: "2022-03-15"
    warranty_type: ProSupport Plus
    warranty_expiry: "2025-03-15"
    vendor_eos_date: "2025-12-31"  # End of sale
    vendor_eosl_date: "2028-12-31" # End of service life
    planned_refresh: "2026-Q2"
    location: DC1-Row3-Rack12-U20
    role: production-database
    criticality: high
    notes: "Extended warranty quote requested for 2025-03-15 to 2026-06-30"

  - hostname: web-prod-05
    model: HPE ProLiant DL380 Gen10
    serial: MXQ1234567
    purchase_date: "2020-06-01"
    warranty_type: HPE Care Pack
    warranty_expiry: "2023-06-01"  # EXPIRED
    vendor_eos_date: "2023-03-31"
    vendor_eosl_date: "2026-03-31"
    planned_refresh: "2025-Q1"     # OVERDUE
    location: DC1-Row1-Rack05-U32
    role: web-frontend
    criticality: medium
    notes: "Third-party support contract with Park Place Technologies"
```

**Automated warranty expiration alerting**:

```bash
#!/bin/bash
# check-warranty-expiry.sh
# Alert on servers with warranty expiring within 90 days

WARRANTY_FILE="/etc/inventory/hardware-lifecycle.yml"
DAYS_WARNING=90
TODAY=$(date +%s)

# Parse YAML (simplified - use yq in production)
yq '.servers[] | select(.warranty_expiry != null)' "${WARRANTY_FILE}" | \
while read -r line; do
  hostname=$(echo "$line" | yq '.hostname')
  expiry=$(echo "$line" | yq '.warranty_expiry')
  expiry_epoch=$(date -d "$expiry" +%s 2>/dev/null)

  if [ -n "$expiry_epoch" ]; then
    days_until=$(( (expiry_epoch - TODAY) / 86400 ))
    if [ $days_until -lt 0 ]; then
      echo "[CRITICAL] ${hostname}: Warranty EXPIRED $((-days_until)) days ago (${expiry})"
    elif [ $days_until -lt $DAYS_WARNING ]; then
      echo "[WARNING] ${hostname}: Warranty expires in ${days_until} days (${expiry})"
    fi
  fi
done
```

---

## Quick Reference: Severity Summary

| Severity | Firmware Lifecycle Findings |
|----------|----------------------------|
| CRITICAL | Decommissioned hardware without secure data wipe; decommissioned servers with BMC credentials intact; NIST 800-88 non-compliant data destruction |
| HIGH | No firmware inventory; BMC/BIOS firmware more than one major version behind; RAID controller below vendor minimum; SSD firmware with known data loss bug; critical CVE unpatched beyond 30-day SLA; no rollback procedure; untested firmware deployed to production; speculative execution mitigations missing; servers past EOSL still in production without risk acceptance; workloads not drained before firmware reboot; cluster quorum violated during maintenance; decommissioned BMC credentials not cleared; vendor EOL dates not tracked; no CVE monitoring for firmware |
| MEDIUM | NIC/HBA firmware not tracked; inventory not automated; inventory update schedule missing; firmware not compared against vendor support matrices; vendor advisories not monitored; update packages not pre-staged; component compatibility matrix not consulted; update runbook missing; pre-update health checks missing; post-update validation missing; emergency patching SLA undefined; firmware package integrity not verified; change management process not followed; monitoring not suppressed during maintenance; hardware refresh cycle undefined; budget not planned in advance; asset disposition not documented; aging hardware not monitored; spare parts not maintained; speculative execution variant tracking incomplete; microcode not loaded via OS package; CVE SLAs not matching software SLAs; firmware scanning not in security assessments |
| LOW | GPU/CPLD/PSU/backplane firmware not tracked; firmware ahead of vendor recommendation; update logs not retained; maintenance window duration not estimated; firmware updates not batched for single reboot; extended warranty not evaluated |
| INFO | Comprehensive automated firmware inventory; canary-based fleet update process; all speculative execution mitigations verified; hardware lifecycle tracked with 12-month refresh planning; NIST 800-88 compliant decommissioning with chain of custody |

---

## References

- NIST SP 800-88 Rev. 1: Guidelines for Media Sanitization
- NIST SP 800-147: BIOS Protection Guidelines
- NIST SP 800-193: Platform Firmware Resiliency Guidelines
- DMTF Redfish API Specification (firmware update service)
- Dell DSA (Dell Security Advisories): [dell.com/support/security](https://www.dell.com/support/security)
- Dell DSU (Dell System Update) User Guide
- HPE Customer Bulletins and Security Advisories
- HPE Service Pack for ProLiant (SPP) documentation
- Supermicro Security Advisories and SUM documentation
- Intel Security Center: [intel.com/security](https://www.intel.com/security)
- AMD Product Security: [amd.com/en/resources/product-security](https://www.amd.com/en/resources/product-security)
- MITRE CVE Database: [cve.mitre.org](https://cve.mitre.org)
- CIS Benchmarks for server operating systems
- Intel Transient Execution Attacks documentation (Spectre, Meltdown, MDS, L1TF, TAA, MMIO Stale Data, Downfall)
- AMD Predictive Store Forwarding, Inception, Zenbleed security advisories
