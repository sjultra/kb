# Raspberry Pi 5 NVMe Crashes — Build, Troubles, Troubleshooting

> **Living document.** Last updated November 11, 2026. This page is updated as the investigation and the fixes progress. See the Current status section for the latest verdict.

## Summary

Five Raspberry Pi 5 nodes run K3s at a retail client's store locations. Each Pi uses a Power-over-Ethernet (PoE) hat for power and an NVMe SSD over PCIe for storage.

One store's Pi crashes every 2–8 hours: the NVMe drive drops off the PCIe bus, followed by K3s timeout errors. The root cause is a drive-model incompatibility — the WD Black SN770 is on the Pi 5 vendor incompatibility list — not bad flash, not power, and not a Pi firmware bug.

Three fixes were applied on November 11, 2026: the Pi bootloader was updated from the May 8, 2025 release to the December 8, 2025 release, the SN770 firmware was updated from 731100WD (built June 8, 2022) to 731120WD (built October 15, 2023), and a PCIe link setting (`dtparm=pciex1_no_l0s`) was added. The drops recurred after all three fixes — four controller-down events on November 11 (09:29, 10:12, 10:20, 10:44 PST) — so the fixes did not hold. The drive is faulty or incompatible with the Pi 5. Replacement with a Pi 5-proven model (the WD Black SN7100, which runs clean at a second store) is approved and is the next step. A 10-minute drop watch monitors until the drive is replaced.

Three more Pis are offline with no remote telemetry. Recovery is remote: the UniFi monitoring tool detects the offline Pi, identifies its switch port, and power-cycles the PoE port to bring it back online. No site visit is needed — remote PoE power-cycle is why the Pis use PoE.

## How the Pis were built

Each store runs a single Raspberry Pi 5 as its K3s node. The Pi is the only control-plane and etcd node for that store's cluster, so a Pi outage takes that store's control plane down (workloads keep running until they need the API).

These Pis are the [FlexPondi Pi5](https://www.portalrobotics.com/flexpondi-pi5.html) edge appliance from [Portal Robotics](https://www.portalrobotics.com/) — a compact appliance that connects each location to the VS Platform control plane for deployment, security, monitoring, and management. The FlexPondi Pi5 is part of the Portal Robotics FlexPondi Toolkit.

- **Hardware:** Raspberry Pi 5, PoE hat (power over the network cable), NVMe SSD on a PCIe M.2 HAT.
- **Storage:** The affected store uses a WD Black SN770 500GB (DRAM-less design, uses Host Memory Buffer). The control store uses a WD Black SN7100 500GB.
- **Software:** Linux with K3s (Kubernetes). The Pis are managed through Salt for remote telemetry and maintenance.
- **Sites:** Five stores — one Pi per store.
- **Power:** PoE, so a hung Pi can be power-cycled remotely by toggling its switch port (see Recovery below).
- **Monitoring:** The [UniFi Monitor](https://github.com/sjultra/unifi-monitor) (open source, SJULTRA) monitors the UniFi network devices, detects when a Pi goes offline, and power-cycles the PoE port to bring it back online.

## Parts

The parts for this build, with purchase links. Prices are as listed on the Central Computer Store on November 11, 2026; availability changes, so confirm before ordering. Where the exact part is not stocked, the in-stock substitute is linked instead.

| Part | Role | Purchase |
| --- | --- | --- |
| Raspberry Pi 5 | The K3s node at each store | — |
| WD Black SN770 500GB (WDS500G3X0E) | The drive in the affected store — **not recommended** (incompatible with Pi 5, see root cause) | Not currently listed at the Central Computer Store; the SN770M 500GB variant is special order (out of stock) |
| WD Black SN7100 500GB (WDS500G4X0E) | The Pi 5-proven replacement drive (runs clean at the control store) | Substitute the in-stock 1TB model: [Central Computer Store — $189.99](https://www.centralcomputer.com/western-digital-wds100t4x0e-wd-black-1tb-sn7100nvme-internal-ssd-pcie-gen-4-m-2-2280-72500mb-s-reads.html) |
| Raspberry Pi PoE+ HAT (Pi 5) | Powers the Pi over the network cable (802.3at PoE+, up to 25W) | Substitute the in-stock Geekworm Pi 5 PoE HAT (X1200-A1): [Central Computer Store — $24.95](https://www.centralcomputer.com/geekworm-raspberry-pi-5-poe-hat-support-802-3af-at-with-pd-trigger-activation-type-c-power-black.html) |
| Cat6 Ethernet patch cable, 7 ft, blue | Connects the Pi to the switch (carries data + PoE power) | [Central Computer Store — $4.95](https://www.centralcomputer.com/cat6-straight-patch-550mhz-utp-cable-7-blue.html) |
| Ubiquiti USW-LITE-8-POE | PoE switch — 8 ports, 52W PoE budget, 4x PoE+ ports; adequate PoE power for Pi 5 nodes | [Central Computer Store — $119.99](https://www.centralcomputer.com/ubiquiti-usw-lite-8-poe-unifi-switch-lite-8-portpoe-switch-8x-rj45-802-3at-poe-52w-max.html) |
| UniFi Dream Machine (UDM) | Cloud gateway / UniFi console for a store | Substitute the in-stock UDM-PRO (1U rackmount): [Central Computer Store — $415.99](https://www.centralcomputer.com/ubiquiti-udm-pro-unifi-dream-machine-pro-1u-rackmount-8-port-switch-10gbps-3-5in-hdd-expansion.html) |
| UniFi Dream Router 7 (UDR7) | Cloud gateway / UniFi console for a store — WiFi 7, 4x 2.5GbE ports (1 PoE), 10G SFP+ WAN | [Central Computer Store — $299.99](https://www.centralcomputer.com/ubiquiti-udr7-us-dream-router-wifi-7-4-port-switch.html) |

**Note on PoE power:** A Pi 5 under load needs an 802.3at (PoE+) port delivering up to 25W. Plain 802.3af ports (15.4W) are not enough for a Pi 5. Confirm the switch PoE budget covers all PoE devices on it (the UDR7 has a 15.4W PoE budget on its single PoE port; the USW-LITE-8-POE has a 52W budget across 4 PoE+ ports).

Example of the switch and mini rack type: the [Central Computer event recap — ASUS Ascent GX10 mini-rack](https://www.centralcomputer.com/blog/post/asus-x-nvidia-x-central-computers-%7C-event-recap#ascent-gx10) shows a Ubiquiti 10G PoE switch in a 10-inch mini rack — an example of the type of switch and mini rack that could be used.

## The troubles

### Affected store Pi: NVMe drops off the PCIe bus

The affected store Pi's NVMe drive repeatedly drops off the PCIe bus. The kernel reports the controller is down and resets it. Each drop is followed within seconds by K3s timeout errors (a downstream symptom, not the cause).

- **Frequency:** 12 controller-down events from November 9–11, 2026, roughly every 2–8 hours — including 4 on November 11 after all three low-cost fixes were applied.
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

- **Affected store Pi:** All three low-cost fixes were applied and verified, but the drops recurred. Four controller-down events on November 11 after the fixes (09:29, 10:12, 10:20, 10:44 PST). The Pi stayed up through all four drops (no reboot). Verdict: the fixes did not hold. The drive is faulty or essentially incompatible with the Pi 5. Replacement with the WD Black SN7100 is approved and is the next step.
- **Drop watch:** A check runs every 10 minutes for new NVMe controller-down events. It stays silent unless a new drop occurs. It continues until the drive is replaced.
- **Control store Pi:** Healthy, zero drops. No action needed.
- **Three offline Pis:** Still offline, no telemetry. Recovery is remote via PoE power-cycle (see Next steps); no site visit needed.

## Next steps

- **Replace the SN770 on the affected store Pi (approved November 11):** The low-cost fixes did not hold — four drops recurred after all three fixes. Replace the SN770 with a Pi 5-proven drive (the WD Black SN7100, as used at the control store).
- **Three offline Pis:** Recover remotely via the UniFi monitor: detect the offline Pi, identify its switch port, power-cycle the PoE port, and confirm the Pi comes back online. No site visit needed — remote PoE power-cycle is why the Pis use PoE.
- **Second node at the affected store:** Confirm what the second node at the affected store is and whether it carries an SN770 (it is not a Pi and is outside the five-Pi scope, but the drive question matters).
- **This page:** Updated as the drop watch verdict lands and as the offline Pis are recovered.

## References

- Pi 5 vendor incompatibility list (Geekworm) — lists the WD Black SN770 as incompatible with the Pi 5.
- OpenZFS discussion #14793 — documents the same controller-down signature for the SN770 on this firmware version.
- WD Black SN770 firmware 731120WD — latest build for the 500GB model (vendor download repository).
- [FlexPondi Pi5](https://www.portalrobotics.com/flexpondi-pi5.html) — the edge appliance these Pis are, from Portal Robotics.
- [UniFi Monitor](https://github.com/sjultra/unifi-monitor) — open-source UniFi network monitoring (SJULTRA).
