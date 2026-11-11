# Raspberry Pi 5 NVMe Crashes — Build, Troubles, Troubleshooting

> **Living document.** Last updated November 11, 2026. This page is updated as the investigation and the fixes progress. See the Current status section for the latest verdict.

## Summary

Five Raspberry Pi 5 nodes run K3s at a retail client's store locations. Each Pi uses a Power-over-Ethernet (PoE) hat for power and an NVMe SSD over PCIe for storage.

One store's Pi crashes every 2–8 hours: the NVMe drive drops off the PCIe bus, followed by K3s timeout errors. The root cause is a drive-model incompatibility — the WD Black SN770 is on the Pi 5 vendor incompatibility list — not bad flash, not power, and not a Pi firmware bug.

Three fixes were applied on November 11, 2026: the Pi bootloader was updated from the May 8, 2025 release to the December 8, 2025 release, the SN770 firmware was updated from 731100WD (built June 8, 2022) to 731120WD (built October 15, 2023), and a PCIe link setting (`dtparm=pciex1_no_l0s`) was added. A 10-minute drop watch monitors for recurrence. If drops recur, the next step is replacing the drive with a Pi 5-proven model (the WD Black SN7100, which runs clean at a second store).

Three more Pis are offline with no remote telemetry. Recovery is remote: the UniFi monitoring tool detects the offline Pi, identifies its switch port, and power-cycles the PoE port to bring it back online. No site visit is needed — remote PoE power-cycle is why the Pis use PoE.

## How the Pis were built

Each store runs a single Raspberry Pi 5 as its K3s node. The Pi is the only control-plane and etcd node for that store's cluster, so a Pi outage takes that store's control plane down (workloads keep running until they need the API).

- **Hardware:** Raspberry Pi 5, PoE hat (power over the network cable), NVMe SSD on a PCIe M.2 HAT.
- **Storage:** The affected store uses a WD Black SN770 500GB (DRAM-less design, uses Host Memory Buffer). The control store uses a WD Black SN7100 500GB.
- **Software:** Linux with K3s (Kubernetes). The Pis are managed through Salt for remote telemetry and maintenance.
- **Sites:** Five stores — one Pi per store.
- **Power:** PoE, so a hung Pi can be power-cycled remotely by toggling its switch port (see Recovery below).

## The troubles

### Affected store Pi: NVMe drops off the PCIe bus

The affected store Pi's NVMe drive repeatedly drops off the PCIe bus. The kernel reports the controller is down and resets it. Each drop is followed within seconds by K3s timeout errors (a downstream symptom, not the cause).

- **Frequency:** 8 controller-down events from November 9–11, 2026, roughly every 2–8 hours.
- **Symptom:** The drive disappears from the PCIe bus, the controller resets, and K3s reports timeouts until the drive comes back.
- **Impact:** The store's K3s control plane is interrupted each time the drive drops.

### Three Pis offline with no telemetry

Three of the five Pis are unreachable and have no remote management enrollment. No telemetry exists for them, so their drive models, firmware versions, and health are unknown.

Recovery does not need a site visit. The Pis use Power-over-Ethernet precisely so they can be power-cycled remotely: the UniFi monitoring tool detects when a Pi goes offline, identifies the Ethernet port on the UniFi switch the Pi is connected to, and power-cycles that PoE port to bring the Pi back online.

### Control store Pi: healthy control case

The control store Pi runs the same OS, kernel, and PoE hat as the affected store but a different drive (WD Black SN7100 500GB) and the current bootloader. It has zero drops in 38+ hours. The drive model (and/or bootloader) is the differentiator.

## Troubleshooting so far

### What the telemetry showed

Telemetry was collected remotely via Salt (kernel logs, drive health, power status).

- **Drive health is clean.** SMART reports zero media errors, 1% used, 100% spare. The flash itself is not bad.
- **Power is clean.** The Pi reports thermal throttling occurred, but no under-voltage history. The under-voltage bit is clear. PoE/power is exonerated.
- **Kernel workarounds were already applied and do not stop the drops.** The settings `nvme_core.default_ps_max_latency_us=0`, `pcie_aspm=off`, and `pcie_port_pm=off` were already in place before the investigation and the drops continued.
- **The drive model is the problem.** The WD Black SN770 is on the Pi 5 vendor incompatibility list. The same controller-down signature is documented for this drive and firmware version elsewhere. The SN770 is a DRAM-less design that relies on Host Memory Buffer (HMB), and this firmware version has a known HMB/power-state problem.

### What was ruled out

- **Bad NAND/flash:** SMART is clean; the drive is not failing.
- **PoE/power:** No under-voltage events; power is stable.
- **Pi firmware bug as the primary driver:** The bootloader was older, but the drive-model incompatibility explains the drops better than a Pi firmware bug alone. The control store Pi (same Pi, different drive) is stable.

## Fixes applied (November 11, 2026)

All three are low-cost fixes, applied in one maintenance window and verified after reboot. If drops recur, the fallback is drive replacement (see Next steps).

### 1. Bootloader updated from May 8, 2025 to December 8, 2025

The Pi bootloader was updated from the May 8, 2025 release to the current release (December 8, 2025) and verified. The old bootloader was about 7 months out of date. The Pi rebooted cleanly and the bootloader reports up to date.

### 2. SN770 firmware updated from 731100WD to 731120WD

The SN770 firmware was updated from 731100WD to 731120WD (the latest build for the 500GB model), applied with `nvme-cli` (`fw-download` + `fw-commit`), then the Pi was rebooted. The new firmware version persisted across the reboot and was verified.

The old firmware (731100WD) was built June 8, 2022 — over four years old at the time of the update. The new firmware (731120WD) was built October 15, 2023.

**Getting the firmware file was itself a problem.** The vendor download host blocks cloud and datacenter networks (it returns an empty reply and presents an expired certificate to those networks), so the file could not be fetched from a cloud VM or a browser. The direct firmware link worked from residential networks. The file was downloaded, verified as a genuine vendor firmware image (correct format header and build references, matching checksum on two separate downloads), and then applied.

**Caveat:** 731120WD is the latest 500GB build, but it is not documented as the fix for this drop issue. The documented HMB fix (731130WD, which caps HMB at 64MB) is for the 2TB model only. Whether 731120WD fixes the Pi 5 PCIe drops is unproven — the drop watch is the verdict.

### 3. PCIe link setting `dtparm=pciex1_no_l0s` added

A PCIe link setting (`dtparm=pciex1_no_l0s`) was added to the Pi boot configuration. This is a documented workaround for drives that drop off the PCIe bus. It is low-risk and reversible (remove the one line and reboot). It was applied in the same window because the firmware alone was unproven and the observation window after the bootloader update alone was only minutes (drops occur every 2–8 hours).

## Current status (November 11, 2026)

- **Affected store Pi:** All three low-cost fixes applied and verified. Firmware is 731120WD (was 731100WD, built June 2022), bootloader is the December 8, 2025 release (was May 8, 2025), the PCIe setting is present, K3s is active, and there are zero controller-down events in the current boot.
- **Drop watch:** A check runs every 10 minutes for new NVMe controller-down events. It stays silent unless a new drop occurs. The first run after the fixes showed no new drops. The next drop, if the fixes did not hold, would be expected roughly by early afternoon on November 11 (2–8 hours after the last drop). A clean 24 hours is stronger evidence.
- **Control store Pi:** Healthy, zero drops. No action needed.
- **Three offline Pis:** Still offline, no telemetry. Recovery is remote via PoE power-cycle (see Next steps); no site visit needed.

## Next steps

- **If a drop recurs on the affected store Pi:** The low-cost fixes did not hold. Replace the SN770 with a Pi 5-proven drive (the WD Black SN7100, as used at the control store).
- **Three offline Pis:** Recover remotely via the UniFi monitor: detect the offline Pi, identify its switch port, power-cycle the PoE port, and confirm the Pi comes back online. No site visit needed — remote PoE power-cycle is why the Pis use PoE.
- **Second node at the affected store:** Confirm what the second node at the affected store is and whether it carries an SN770 (it is not a Pi and is outside the five-Pi scope, but the drive question matters).
- **This page:** Updated as the drop watch verdict lands and as the offline Pis are recovered.

## References

- Pi 5 vendor incompatibility list (Geekworm) — lists the WD Black SN770 as incompatible with the Pi 5.
- OpenZFS discussion #14793 — documents the same controller-down signature for the SN770 on this firmware version.
- WD Black SN770 firmware 731120WD — latest build for the 500GB model (vendor download repository).
