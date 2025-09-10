# RHEL9 Turnkey Golden Image – Operator & DevOps Cheat Sheet

## 1. Pipeline Basics
- File: `azure-pipelines-rhel9.yml`
- Stages: Validate → Build (Gen2 default, Gen1 optional) → Compliance (OpenSCAP) → Smoke Test → Publish
- Trigger: manual only (no CI triggers by default)
- Parameters:
  - `useGen1`: set `true` to build Gen1 in addition to Gen2
  - `location`: Azure region to build and test in
  - `sigResourceGroup`: Resource group of Shared Image Gallery
  - `sigName`: Shared Image Gallery name
  - `imageDefinition`: target image definition
  - `imageVersion`: version number (increment per release)

## 2. Catalog & Staging
- File: `linux/catalog.json`
- Purpose: Define agent installers/tools to stage into image
- Each entry requires:
  - `id`, `version`, `artifact` (HTTPS URL), `sha256`, `stage` (true/false)
- Policy guard will fail build if:
  - URL not HTTPS
  - SHA256 is missing/placeholder
  - ID not in allowlist
- Staged files land in: `/opt/stage/Tools/<id>/<version>/`

## 3. Scripts Overview
- `00_prereqs.sh`: updates, installs jq/curl/openssl/unzip, configures sshd, enables rsyslog & auditd
- `10_hardening.sh`: SELinux enforcing, firewalld, crypto policies, sysctl hardening
- `20_catalog_prestage.sh`: loops through `catalog.json` and downloads/validates artifacts
- `90_cleanup.sh`: cleans yum caches, resets machine-id
- `99_deprovision.sh`: runs `waagent -deprovision+user -force`

## 4. Post-Deploy (Ansible)
- Run `ansible/postdeploy.yml` after VM creation
- Includes roles:
  - `agent_enroll`: enroll CrowdStrike, Tanium, Splunk, etc. using AKV-sourced secrets
  - `certlc_runner`: Microsoft CERTLC pattern for certificate rotation

### Agent Enrollment
- Controlled via `group_vars/all.yml`
- Example:
  ```yaml
  cs_cid: "<CID_FROM_AKV>"
  tanium_server: "tanium01.molina.local"
  splunk_deployment_server: "splunkdeploy.molina.local"
  splunk_boot_pass: "<BOOTSTRAP_PASSWORD_FROM_AKV>"
  ```

### Certificate Rotation (CERTLC)
- Enable with `enable_certlc: true` in `group_vars/all.yml`
- Requires AKV + Managed Identity
- Variables:
  ```yaml
  certlc_zip_url: "https://github.com/Azure/certlc/raw/.../script_for_not_supported_ARC_on_Linux_distro.zip"
  certlc_zip_sha256: "<SHA256>"
  akv_name: "molina-akv"
  cert_object_name: "service-tls-pem"
  target_mode: "pem"    # or "pair"
  cert_restart_service: "nginx"
  certlc_poll_interval: "1h"
  ```
- Systemd timer: `certlc-run.timer` runs hourly by default

## 5. Validation & Compliance
- `tests/linux/validate_compliance.sh`: runs OpenSCAP minimal profile inside test VM
- `tests/linux/validate_image_smoke.sh`: boots test VM, verifies staged tools, auditd, SELinux

## 6. Common Commands
- Show latest image in SIG:
  ```bash
  az sig image-version list -g <RG> --gallery-name <SIG> --gallery-image-definition <DEF> -o table
  ```
- Tail certlc logs:
  ```bash
  journalctl -u certlc-run.service
  less /var/log/certlc/fallback.log
  ```
- Force immediate cert refresh:
  ```bash
  systemctl start certlc-run.service
  ```

## 7. Troubleshooting
- Build fails → check policy-guard output (SHA/HTTPS/allowlist)
- VM won’t boot → review smoke test logs (`validate_image_smoke.sh`)
- Cert not updating → ensure AKV RBAC (`Key Vault Secrets User` for VM MSI) and check `/etc/certlc/certlc.env`
- Staged files missing → verify `catalog.json` and SHA values

## 8. Key Security Notes
- No secrets in golden images (staging only)
- SHA-256 validation enforced
- SELinux enforcing, auditd + rsyslog enabled
- Certificates rotated runtime only, never baked
- Machine-id reset during cleanup to avoid duplicate identities

---
**Remember:** Always increment `imageVersion` per release, and retire old images from SIG with lifecycle policies.
