## 📌 Incident Overview

This repository documents the root-cause diagnosis, audit trail, and command-line recovery of a failed offline system update on a Fedora/RHEL-based Linux system (Nobara/Fedora 44 kernel stack) that triggered a boot-time recovery rollback and locked package management services.

## 🔍 Root Cause Analysis & Initial Triage

- **Interrupted Offline Update & Fallback Rollback:** An aborted offline system update (`dnf5` Transaction ID `10`) triggered the boot-time *System Update Recovery* rollback dialog (`"The previous update did not finish"`), leaving behind active lock files (`/var/lib/rpm/.rpm.lock` and `/var/lib/dnf/lock`) and an inconsistent RPM header database.
- **Failed Offline Update Daemon:** `nobara-updater-prepare.service` exited with an error state due to unresolved dependency conflicts across kernel and audio/system packages (`kernel-core`, `pipewire`, `wireplumber`, and `nss`).
- **Initial Triage:** Standard GUI updater retries, `dnf upgrade` commands, and systemd update service restarts failed immediately against the locked RPM database, requiring manual lock clearance and full repository state synchronization.

## 🛠️ Step-by-Step CLI Remediation

1. **Cleared Stale Package Locks & Rebuilt the RPM Database:**
   Removed orphaned lock files and reconstructed the local RPM header index to restore package integrity:
   ```bash
   sudo rm -f /var/lib/rpm/.rpm.lock /var/lib/dnf/lock
   sudo rpm --rebuilddb
   ```

2. **Halted the Failed Offline Update Service:**
   Stopped the stalled systemd preparation unit so manual package synchronization could proceed safely:
   ```bash
   sudo systemctl stop nobara-updater-prepare.service
   ```

3. **Synchronized Distribution Packages & Resolved Dependency Conflicts:**
   Executed a full distribution synchronization (`dnf5` Transaction ID `16`) to reconcile mismatched package versions and remove orphaned kernel artifacts:
   ```bash
   sudo dnf distro-sync --allowerasing
   ```

## ✅ Verification & Repository Artifacts

- **System Restored:** Eliminated the boot-time recovery loop and restored clean `dnf5` and `systemd` update workflows without data loss or OS reinstallation.
- **`dnf5-failed-transaction-10.txt`:** Sanitized `dnf5 history info 10` audit log showing the initial failed transaction (`Status: Error`).
- **`dnf5-recovery-log.txt`:** Sanitized `dnf5 history info 16` audit log confirming clean package upgrades and replacements (`Status: Ok`).
