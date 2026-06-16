# Engineering Lab Log: Full-Day Analysis
**Date:** June 12, 2026  
**Subject:** Hardware Failure Analysis and System Recovery  

---

## 1. Executive Summary
This document serves as the formal record for the diagnostic and recovery operations performed on the AMD FX-8350 bare-metal server on June 12, 2026. The objective was to troubleshoot persistent kernel panics and storage controller lock-ups that prevented the initialization of the MicroCloud cluster. The day concluded with a fully stabilized hardware layer and a clear path for deployment via direct LXD orchestration.

---

## 2. Chronological Analysis
### Phase 1: Symptom Identification (Early-Day)
* **Initial State:** System was operational but failed during `sudo microcloud init`.
* **Symptom:** Terminal output displayed `context deadline exceeded` and `no such file or directory` (socket errors).
* **Deep Analysis:** The kernel logs (retrieved via `dmesg`) indicated `ata5.00: exception Emask 0x10... frozen`. This suggested the storage controller was physically dropping communication with the SSD during I/O intensive database write operations.

### Phase 2: Hardware Diagnosis (Mid-Day)
* **Visual Inspection:** The system chassis was opened to inspect the SATA interface.
* **Controller Identification:** The system was utilizing SATA Port 5 (top block, SATA III), which was experiencing signal noise or interface timing issues on the legacy SB950 chipset.
* **Corrective Action:** Performed physical migration of the SATA data cable from the upper block (SATA III) to the lower block (SATA Port 4, SATA II).
* **Rationale:** By shifting to the SATA II controller pathway, we bypassed the specific channel causing signal noise and utilized a more stable, albeit slightly slower, interface that matched the SSD's capabilities without NCQ (Native Command Queuing) overhead triggering the legacy driver faults.

[Image of typical SATA architecture and disk connection]

### Phase 3: Software & Kernel Cleanup (Late-Day)
* **Validation:** Post-cable migration, `smartctl` reported the drive as `PASSED`, and subsequent `journalctl -k` logs showed no further `Emask` or `frozen` errors. 
* **The "AppArmor" Conflict:** With hardware stable, a secondary software issue appeared—AppArmor logs showed the corrupted MicroCloud snap daemons were in a permission-denied loop (`operation="open" class="file" denied_mask="r"`).
* **Resolution:** Decided to pivot from the failing multi-node cluster architecture (MicroCloud) to a robust single-node hypervisor architecture (LXD). The snap environment was purged, and LXD was initialized natively.

---

## 3. Root Cause Analysis (RCA)
| Issue | Root Cause | Resolution |
| :--- | :--- | :--- |
| **I/O Freezes** | Legacy AMD chipset timing desync with SATA III NCQ requests. | Moved to SATA Port 4 (SATA II channel) to force stable signaling. |
| **Service Hangs** | Corrupted dqlite database (caused by abrupt power-off). | Purged broken snaps and re-initialized engine from scratch. |
| **Log Flooding** | AppArmor blocking daemons in a failure loop. | Clean removal of broken snap profiles. |

---

## 4. Current System Baseline (End of Day)
* **Hardware:** Healthy (SSD verified at `ata2` channel).
* **Network:** Interface `enp3s0` active at `192.168.1.8`.
* **Orchestration:** Clean LXD state (ready for service deployment).
* **Storage Path:** `/dev/sda` partition `sda3` correctly mounted.

---

## 5. Deployment Readiness
The system is now primed for the following operations on June 13, 2026:
1. Initialize standard LXD container runtime.
2. Provision RabbitMQ broker node.
3. Establish messaging queue connectivity for the Saga pattern workflow.