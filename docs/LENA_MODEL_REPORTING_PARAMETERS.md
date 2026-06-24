# Reporting Parameters — 5G-LENA (NR Module)

---

## Introduction

This document presents the capabilities of the 5G-LENA module by identifying the reporting parameters (KPIs) that can be extracted from each protocol layer. It provides an overview of the available measurements and reporting mechanisms implemented in the current 5G-LENA codebase under `src/nr/`.

The purpose of this document is to help developers understand what information is available from each layer and how it can be used for monitoring, analytics, and O-RAN xApp development.

In the next step, this document will demonstrate how to develop a custom xApp by extending the reporting framework to extract and report custom parameters that are not available by default.

> **Note:** in 5G

---

## Status Definitions

| Status | Meaning |
|--------|---------|
| **IMPLEMENTABLE** | A dedicated trace source, public API, or helper-connected file sink exists in the simulator; the metric can be read directly (some require `Config::Connect` or a non-default helper call). |
| **DERIVABLE** | No native 3GPP PM counter exists, but the value can be computed from other simulator outputs (e.g. Δbytes/Δt, epoch aggregation, post-processing of trace files). |
| **NOT_SUPPORTED** | No trace, API, or reliable derivation path exists in the current NR module. |

---

## Summary

| Layer | IMPLEMENTABLE | DERIVABLE | NOT_SUPPORTED | Total |
|-------|:-------------:|:---------:|:-------------:|:-----:|
| PHY | 24 | 2 | 0 | 26 |
| MAC | 6 | 2 | 0 | 8 |
| RLC | 9 | 2 | 0 | 11 |
| PDCP | 7 | 2 | 0 | 9 |
| RRC | 14 | 0 | 0 | 14 |
| E2 / O-RAN | 6 | 2 | 0 | 8 |
| **Total** | **66** | **10** | **0** | **76** |

### Corrections vs. source doc

1. **UL SINR (`UlSinrTrace`)** — trace exists on `NrGnbPhy` but is **not** wired by `EnableUlPhyTraces()`; requires manual `Config::Connect` (sink: `NrPhyRxTrace::UlSinrTraceCallback`).
2. **`EnableTransportBlockTrace()`** — connects **DL only** (`ReportDownlinkTbSize`); UL TB requires separate connect to `ReportUplinkTbSize`.
3. **E2 PDCP volume counters** — `GetDlTxData()` / `GetDlRxData()` are read in `BuildRicIndicationMessageCuUp()`, but values passed to `AddPdcpUePmItem()` come from **RLC** `GetTxPacketsInReportingPeriod()` / `GetTxBytesInReportingPeriod()` (mislabelled in KPM names).

---

## PHY Layer

| # | 3GPP-style KPI | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|----------------|:------:|---------|-------|----------------------|-------------|
| 1 | **L1M.RS-SINR** (DL data) | IMPLEMENTABLE | `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc`, `model/nr-ue-phy.cc` | `NrUePhy`, `NrPhyRxTrace` | TraceSource `DlDataSinr`; sink `NrPhyRxTrace::DlDataSinrCallback`; enable `NrHelper::EnableDlDataPhyTraces()` | Per-TB DL data SINR is traced at UE PHY and written to `DlDataSinr.txt` when traces are enabled. |
| 2 | **L1M.RS-SINR** (DL control) | IMPLEMENTABLE | `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc`, `model/nr-ue-phy.cc` | `NrUePhy`, `NrPhyRxTrace` | TraceSource `DlCtrlSinr`; sink `NrPhyRxTrace::DlCtrlSinrCallback`; enable `NrHelper::EnableDlCtrlPhyTraces()` | DL control-channel SINR is traced at UE PHY and written to `DlCtrlSinr.txt`. |
| 3 | **L1M.RS-SINR** (UL) | IMPLEMENTABLE | `model/nr-gnb-phy.cc`, `helper/nr-phy-rx-trace.cc` | `NrGnbPhy`, `NrPhyRxTrace` | TraceSource `UlSinrTrace` (fired in `NrGnbPhy` SRS processing); sink `NrPhyRxTrace::UlSinrTraceCallback` (**manual** `Config::Connect`) | UL SINR spectrum values are available via trace, but `EnableUlPhyTraces()` does not connect this source automatically. |
| 4 | **RRQ.RSRP** | IMPLEMENTABLE | `model/nr-ue-phy.cc` | `NrUePhy` | TraceSource `ReportRsrp`; callback via `Config::Connect` | RSRP per cell/RNTI is emitted when RS measurements are processed at UE PHY. |
| 5 | **RRQ.RSRQ** | IMPLEMENTABLE | `model/nr-ue-phy.cc` | `NrUePhy` | TraceSource `ReportUeMeasurements`; `NrUePhy::ReportUeMeasurements()` | Combined RSRP/RSRQ measurement report is traced periodically from UE PHY. |
| 6 | **L1M.PathLoss** (channel) | IMPLEMENTABLE | `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc` | `SpectrumChannel`, `NrPhyRxTrace` | TraceSource `PathLoss` on channel; sink `NrPhyRxTrace::PathlossTraceCallback`; `NrHelper::EnablePathlossTraces()` | Large-scale pathloss between TX/RX spectrum models is logged to `UlPathlossTrace.txt` / `DlPathlossTrace.txt`. |
| 7 | **L1M.PathLoss** (DL ctrl/data per UE) | IMPLEMENTABLE | `model/nr-spectrum-phy.cc`, `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc` | `NrSpectrumPhy`, `NrPhyRxTrace` | TraceSources `DlCtrlPathloss`, `DlDataPathloss`; `NrHelper::EnableDlCtrlPathlossTraces()` / `EnableDlDataPathlossTraces()` | Per-UE instantaneous pathloss for DL control/data is traced after enabling flags on `NrSpectrumPhy`. |
| 8 | **L1M.SNR** (per TB) | IMPLEMENTABLE | `model/nr-spectrum-phy.cc` | `NrSpectrumPhy` | TraceSource `DlDataSnrTrace`; manual `Config::Connect` | Per transport-block SNR is available as a spectrum-PHY trace source. |
| 9 | **DRB.UEThpDl** (PHY TB) | DERIVABLE | `model/nr-ue-phy.cc`, `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc` | `NrUePhy`, `NrPhyRxTrace` | TraceSources `ReportDownlinkTbSize`, `ReportUplinkTbSize`; `NrHelper::EnableTransportBlockTrace()` (DL only) | TB byte counts are traced at UE PHY; throughput requires summing ΔTB sizes over a time window (UL needs manual connect). |
| 10 | **L1M.RxBytes** / packet trace | IMPLEMENTABLE | `model/nr-spectrum-phy.cc`, `helper/nr-phy-rx-trace.cc` | `NrSpectrumPhy`, `NrPhyRxTrace` | TraceSources `RxPacketTraceUe`, `RxPacketTraceGnb`; sinks `RxPacketTraceUeCallback`, `RxPacketTraceGnbCallback` | Each received TB triggers `RxPacketTraceParams` written to `RxPacketTrace.txt` when DL/UL PHY traces are enabled. |
| 11 | **L1M.TBler** (simulated) | IMPLEMENTABLE | `model/nr-phy-mac-common.h`, `model/nr-spectrum-phy.cc` | `RxPacketTraceParams`, `NrSpectrumPhy` | Fields `m_tbler`, `m_corrupt` in `RxPacketTraceParams` via `RxPacketTraceUe`/`RxPacketTraceGnb` | Error-model TBLER and corruption flag are populated per TB in the packet trace structure. |
| 12 | **L1M.CQI / RI / MCS** | IMPLEMENTABLE | `model/nr-ue-phy.cc` | `NrUePhy` | TraceSource `CqiFeedbackTrace`; manual `Config::Connect` | Wideband CQI, MCS, and rank index are reported on DL CQI generation. |
| 13 | **L1M.TxPower** PUSCH | IMPLEMENTABLE | `model/nr-ue-power-control.cc`, `model/nr-ue-phy.cc` | `NrUePowerControl`, `NrUePhy` | TraceSource `ReportPuschTxPower`; attribute `EnableUplinkPowerControl` on `NrUePhy` | PUSCH TX power is traced when uplink power control is enabled on UE PHY. |
| 14 | **L1M.TxPower** PUCCH | IMPLEMENTABLE | `model/nr-ue-power-control.cc` | `NrUePowerControl` | TraceSource `ReportPucchTxPower`; manual connect | PUCCH TX power is computed and traced by the UE power-control module. |
| 15 | **L1M.TxPower** SRS | IMPLEMENTABLE | `model/nr-ue-power-control.cc` | `NrUePowerControl` | TraceSource `ReportSrsTxPower`; manual connect | SRS TX power is traced when SRS is transmitted with power control enabled. |
| 16 | **RRU.PrbUsedDl** (utilization) | IMPLEMENTABLE | `model/nr-gnb-phy.cc` | `NrGnbPhy` | `GetPrbUtilization()` | Returns fractional DL PRB utilization averaged over recent slots (simulation model). |
| 17 | **RRU.PrbAvail** / slot resources | IMPLEMENTABLE | `model/nr-gnb-phy.cc` | `NrGnbPhy` | TraceSources `SlotDataStats`, `SlotCtrlStats`; manual `Config::Connect` | Per-slot used/available RBs and active UE count are emitted after each DL allocation. |
| 18 | **RRU.PrbUsedDl** (per-RB map) | IMPLEMENTABLE | `model/nr-gnb-phy.cc` | `NrGnbPhy` | TraceSource `RBDataStats`; manual `Config::Connect` | Per-symbol RB occupancy bitmap is traced from gNB PHY scheduling. |
| 19 | **gNB activity factor** | IMPLEMENTABLE | `model/nr-gnb-phy.cc` | `NrGnbPhy` | `CalculateActivityFactor()` | Returns a 0–1 activity metric derived from PRB utilization and scheduling state. |
| 20 | **Antenna port power** (CCC) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc`, `model/nr-gnb-phy.cc`, `model/nr-cb-type-one-sp.cc` | `NrGnbNetDevice`, `NrGnbPhy`, `NrCbTypeOneSp` | `GetPortPower()`, `SetPortPower()`, `GetPortPowerScaling()` | O-RAN/CCC fork exposes per-port mask and PHY scaling factor for antenna reconfiguration. |
| 21 | **Average TX power** (antenna) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `GetAveragePower()` (returns instantaneous watts from `SampleTransmitPower()`) | CCC power model samples total gNB power every 100 ms; API name says "average" but returns current watts. |
| 22 | **Energy consumption** | IMPLEMENTABLE | `model/nr-gnb-phy.cc`, `model/nr-ue-phy.cc`, `helper/nr-helper.cc` | `NrGnbPhy`, `NrUePhy`, `NrHelper` | `GetTotalEnergyConsumption()`; `NrHelper::StartEnergyMonitoring()` → `LogEnergyToFile()` | Integrated energy (J) is accumulated in PHY and exported to `EnergyConsumption_Cell_<id>.csv`. |
| 23 | **Instantaneous power** | IMPLEMENTABLE | `model/nr-gnb-phy.cc`, `model/nr-ue-phy.cc`, `helper/nr-helper.cc` | `NrGnbPhy`, `NrUePhy`, `NrHelper` | `GetCurrentPowerConsumption()`; read in `NrHelper::CalculateAveragePowerPerCell()` | PHY-level instantaneous power (W) is computed from activity/PRB model separate from CCC port-power model. |
| 24 | **L1M.RSSI / SNR per chunk** | IMPLEMENTABLE | `model/nr-interference.cc` | `NrInterference` | TraceSources `RssiPerProcessedChunk`, `SnrPerProcessedChunk`; manual connect | Per-chunk interference RSSI/SNR is traced during spectrum processing. |
| 25 | **PSD** | IMPLEMENTABLE | `model/nr-ue-phy.cc` | `NrUePhy` | TraceSource `ReportPowerSpectralDensity`; manual connect | UE transmit power spectral density is traced per slot/symbol. |
| 26 | PHY ctrl messages | IMPLEMENTABLE | `model/nr-gnb-phy.cc`, `model/nr-ue-phy.cc`, `helper/nr-helper.cc`, `helper/nr-phy-rx-trace.cc` | `NrGnbPhy`, `NrUePhy`, `NrPhyRxTrace` | TraceSources `GnbPhyRxedCtrlMsgsTrace`, `GnbPhyTxedCtrlMsgsTrace`, `UePhyRxedCtrlMsgsTrace`, `UePhyTxedCtrlMsgsTrace`, `UePhyRxedDlDciTrace`, `UePhyTxedHarqFeedbackTrace`; `EnableGnbPhyCtrlMsgsTraces()`, `EnableUePhyCtrlMsgsTraces()` | PHY control-message traces are connected by default `EnableTraces()` and written to `Rxed/Txed*PhyCtrlMsgsTrace.txt`. |

---

## MAC Layer

| # | 3GPP-style KPI | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|----------------|:------:|---------|-------|----------------------|-------------|
| 1 | **RRU.PrbUsedDl** (per allocation) | IMPLEMENTABLE | `model/nr-gnb-mac.cc`, `helper/nr-mac-scheduling-stats.cc` | `NrGnbMac`, `NrMacSchedulingStats` | TraceSource `DlScheduling`; sink `NrMacSchedulingStats::DlSchedulingCallback`; `NrHelper::EnableDlMacSchedTraces()` | Each DL scheduling decision logs RB/TB parameters to `NrDlMacStats.txt`. |
| 2 | **RRU.PrbUsedUl** | IMPLEMENTABLE | `model/nr-gnb-mac.cc`, `helper/nr-mac-scheduling-stats.cc` | `NrGnbMac`, `NrMacSchedulingStats` | TraceSource `UlScheduling`; sink `NrMacSchedulingStats::UlSchedulingCallback`; `NrHelper::EnableUlMacSchedTraces()` | Each UL scheduling decision is traced to `NrUlMacStats.txt`. |
| 3 | **MAC.UEThpDl** (instantaneous) | DERIVABLE | `helper/nr-mac-scheduling-stats.cc`, `model/nr-phy-mac-common.h` | `NrMacSchedulingStats`, `NrSchedulingCallbackInfo` | Field `m_tbSize` in `NrSchedulingCallbackInfo` from `DlScheduling` trace | MAC throughput is obtained by aggregating scheduled `TbSize` over a time window from MAC stats files. |
| 4 | **L1M.MCS** DL/UL | IMPLEMENTABLE | `model/nr-gnb-mac.cc`, `model/nr-phy-mac-common.h` | `NrGnbMac`, `NrSchedulingCallbackInfo` | Field `m_mcs` in scheduling callback via `DlScheduling` / `UlScheduling` | MCS index is recorded on every MAC scheduling trace entry. |
| 5 | **HARQ.DL.Fail** / feedback | IMPLEMENTABLE | `model/nr-gnb-mac.cc` | `NrGnbMac` | TraceSource `DlHarqFeedback`; `NrGnbMac::DoDlHarqFeedback()`; manual connect | DL HARQ ACK/NACK outcomes are traced when HARQ feedback is processed at gNB MAC. |
| 6 | **MAC.SR** (scheduling request) | IMPLEMENTABLE | `model/nr-gnb-mac.cc` | `NrGnbMac` | TraceSource `SrReq`; manual connect | Scheduling-request events from UE are traced at gNB MAC. |
| 7 | **RACH** timeout | IMPLEMENTABLE | `model/nr-ue-mac.cc` | `NrUeMac` | TraceSource `RaResponseTimeout`; `NrUeMac::RaResponseTimeout()`; manual connect | RA response timeout is traced when contention-based or non-contention RA fails. |
| 8 | MAC ctrl messages | IMPLEMENTABLE | `model/nr-gnb-mac.cc`, `model/nr-ue-mac.cc`, `helper/nr-helper.cc` | `NrGnbMac`, `NrUeMac` | TraceSources `GnbMacRxedCtrlMsgsTrace`, `GnbMacTxedCtrlMsgsTrace`, `UeMacRxedCtrlMsgsTrace`, `UeMacTxedCtrlMsgsTrace`; `EnableGnbMacCtrlMsgsTraces()`, `EnableUeMacCtrlMsgsTraces()` | MAC control messages are traced to `Rxed/Txed*MacCtrlMsgsTrace.txt` when enabled. |

---

## RLC Layer

| # | 3GPP-style KPI | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|----------------|:------:|---------|-------|----------------------|-------------|
| 1 | **DRB.RlcSduVolumeDl** (TX) | IMPLEMENTABLE | `model/nr-rlc.cc`, `helper/nr-bearer-stats-simple.h` | `NrRlc`, `NrBearerStatsSimple` | TraceSource `TxPDU`; `NrHelper::EnableRlcSimpleTraces()` | DL RLC TX PDU bytes are logged to `NrDlTxRlcStats.txt`. |
| 2 | **DRB.RlcSduVolumeDl** (RX) | IMPLEMENTABLE | `model/nr-rlc.cc`, `helper/nr-bearer-stats-simple.h` | `NrRlc`, `NrBearerStatsSimple` | TraceSource `RxPDU`; `NrHelper::EnableRlcSimpleTraces()` | DL RLC RX PDU bytes are logged to `NrDlRxRlcStats.txt`. |
| 3 | **DRB.RlcSduVolumeUl** | IMPLEMENTABLE | `model/nr-rlc.cc`, `helper/nr-bearer-stats-simple.h` | `NrRlc`, `NrBearerStatsSimple` | TraceSources `TxPDU`, `RxPDU` (UL bearers) | UL RLC TX/RX volumes are logged to `NrUlTxRlcStats.txt` / `NrUlRxRlcStats.txt`. |
| 4 | **DRB.RlcSduDelayDl** | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetDlDelay()`, `GetDlDelayStats()`; `NrHelper::EnableRlcE2eTraces()` | End-to-end DL RLC delay per IMSI/LCID is computed and written to `NrDlRlcStatsE2E.txt`. |
| 5 | **DRB.RlcSduDelayUl** | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetUlDelay()`, `GetUlDelayStats()`; `NrHelper::EnableRlcE2eTraces()` | End-to-end UL RLC delay is exported in `NrUlRlcStatsE2E.txt`. |
| 6 | **DRB.RlcSduBitrateDl** | DERIVABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetDlTxData(imsi, lcid)` divided by calculator epoch (`SetEpoch()`) | Bitrate is not a native counter; divide byte volume by the configured stats epoch duration. |
| 7 | TX PDU count | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.h` | `NrBearerStatsCalculator` | `GetDlTxPackets()`, `GetUlTxPackets()` | Per-bearer TX PDU counts are maintained by the bearer stats calculator. |
| 8 | RX PDU count | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.h` | `NrBearerStatsCalculator` | `GetDlRxPackets()`, `GetUlRxPackets()` | Per-bearer RX PDU counts are available from the calculator API. |
| 9 | PDU size distribution | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.h` | `NrBearerStatsCalculator` | `GetDlPduSizeStats()`, `GetUlPduSizeStats()` | Returns avg/min/max/std of PDU sizes per bearer. |
| 10 | RLC drops | IMPLEMENTABLE | `model/nr-rlc.cc` | `NrRlc` | TraceSource `TxDrop`; manual connect | Dropped RLC SDUs are traced when the transmit queue drops packets. |
| 11 | RLC TX in E2 period | IMPLEMENTABLE | `model/nr-rlc.h`, `model/nr-gnb-net-device.cc` | `NrRlc`, `NrGnbNetDevice` | `GetTxPacketsInReportingPeriod()`, `GetTxBytesInReportingPeriod()`; used in `BuildRicIndicationMessageCuUp()` | RLC counters over the E2 reporting window are read directly from DRB RLC instances. |

---

## PDCP Layer

| # | 3GPP-style KPI | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|----------------|:------:|---------|-------|----------------------|-------------|
| 1 | **DRB.PdcpSduVolumeDl** (TX) | IMPLEMENTABLE | `model/nr-pdcp.cc`, `helper/nr-bearer-stats-simple.h` | `NrPdcp`, `NrBearerStatsSimple` | TraceSource `TxPDU`; `NrHelper::EnablePdcpSimpleTraces()` | DL PDCP TX SDU bytes are logged to `NrDlPdcpTxStats.txt`. |
| 2 | **DRB.PdcpSduVolumeDl** (RX) | IMPLEMENTABLE | `model/nr-pdcp.cc`, `helper/nr-bearer-stats-simple.h` | `NrPdcp`, `NrBearerStatsSimple` | TraceSource `RxPDU`; `NrHelper::EnablePdcpSimpleTraces()` | DL PDCP RX SDU bytes are logged to `NrDlPdcpRxStats.txt`. |
| 3 | **DRB.PdcpSduVolumeUl** | IMPLEMENTABLE | `model/nr-pdcp.cc`, `helper/nr-bearer-stats-simple.h` | `NrPdcp`, `NrBearerStatsSimple` | TraceSources `TxPDU`, `RxPDU` (UL) | UL PDCP volumes are logged to `NrUlPdcpTxStats.txt` / `NrUlPdcpRxStats.txt`. |
| 4 | **DRB.PdcpSduDelayDl** | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetDlDelay()`; `NrHelper::EnablePdcpE2eTraces()` | End-to-end DL PDCP delay per IMSI/LCID is in `NrDlPdcpStatsE2E.txt` and via API. |
| 5 | **DRB.PdcpSduDelayUl** | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetUlDelay()`; `NrHelper::EnablePdcpE2eTraces()` | End-to-end UL PDCP delay is exported in `NrUlPdcpStatsE2E.txt`. |
| 6 | **DRB.UEThpDl** (PDCP throughput) | DERIVABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `BuildGUICuUp()` — `(txBytes − prevTxBytes) / (0.1 s × 1000)` Mbps | PDCP throughput is computed from successive `GetDlTxData()` samples, not a native counter. |
| 7 | **DRB.PdcpSduBitrateDl** | DERIVABLE | `helper/nr-bearer-stats-calculator.cc` | `NrBearerStatsCalculator` | `GetDlTxData(imsi, lcid)` / epoch from `SetEpoch()` | Bitrate is derived by dividing TX byte volume by the stats-calculator epoch length. |
| 8 | PDCP packet counts | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.h` | `NrBearerStatsCalculator` | `GetDlTxPackets()`, `GetDlRxPackets()`, `GetUlTxPackets()`, `GetUlRxPackets()` | Per-bearer PDU counts are maintained by the PDCP stats calculator. |
| 9 | Serving cell per bearer | IMPLEMENTABLE | `helper/nr-bearer-stats-calculator.h` | `NrBearerStatsCalculator` | `GetDlCellId()`, `GetUlCellId()` | Serving cell ID per bearer is stored when bearers are created/reconfigured. |

---

## RRC Layer

| # | 3GPP-style KPI / event | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|------------------------|:------:|---------|-------|----------------------|-------------|
| 1 | **RRC.ConnEstabSucc** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrGnbRrc` | TraceSource `ConnectionEstablished`; manual `Config::Connect` | Fired on successful RRC connection establishment at UE and gNB. |
| 2 | **RRC.ConnRel** | IMPLEMENTABLE | `model/nr-gnb-rrc.cc` | `NrGnbRrc` | TraceSource `NotifyConnectionRelease` | Fired when a UE context is released at the gNB. |
| 3 | **RRC.ConnEstabFail** | IMPLEMENTABLE | `model/nr-ue-rrc.cc` | `NrUeRrc` | TraceSource `ConnectionTimeout`; `NrUeRrc::ConnectionTimeout()` | Fired when T300 expires without successful connection. |
| 4 | **MM.MeasRep** / **RRQ.RSRP** in report | IMPLEMENTABLE | `model/nr-gnb-rrc.cc`, `model/nr-rrc-sap.h` | `NrGnbRrc`, `NrUeManager` | TraceSource `RecvMeasurementReport`; `NrUeManager::RecvMeasurementReport()` | Measurement reports containing `rsrpResult` / `rsrqResult` are traced at gNB RRC. |
| 5 | **HO.Att**, **HO.Succ** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrGnbRrc` | TraceSources `HandoverStart`, `HandoverEndOk` | Handover start and successful completion are traced on UE and gNB. |
| 6 | **HO.Fail** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrGnbRrc` | TraceSources `HandoverEndError` (UE), `HandoverFailureNoPreamble`, `HandoverFailureMaxRach`, `HandoverFailureLeaving`, `HandoverFailureJoining` (gNB) | Multiple handover-failure causes are exposed as separate RRC trace sources. |
| 7 | **RRC.State** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrUeManager` | TraceSource `StateTransition` | Every UE RRC state change is traced with IMSI/RNTI/old/new state. |
| 8 | **RRC.RLF** | IMPLEMENTABLE | `model/nr-ue-rrc.cc` | `NrUeRrc` | TraceSources `RadioLinkFailure`, `PhySyncDetection`; `NrUeRrc::RadioLinkFailureDetected()` | Radio link failure and PHY out-of-sync detection are traced at UE RRC. |
| 9 | **RRC.DRB.Estab** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrUeManager` | TraceSource `DrbCreated` | Fired when a DRB is created for a UE. |
| 10 | **RRC.SRB.Estab** | IMPLEMENTABLE | `model/nr-ue-rrc.cc` | `NrUeRrc` | TraceSource `Srb1Created` | Fired when SRB1 is established during connection setup. |
| 11 | **RACH** success/fail | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-ue-mac.cc` | `NrUeRrc`, `NrUeMac` | TraceSources `RandomAccessSuccessful`, `RandomAccessError`; MAC `RaResponseTimeout` | RA success/failure is traced at UE RRC; timeout also available at UE MAC. |
| 12 | **SIB/MIB** received | IMPLEMENTABLE | `model/nr-ue-rrc.cc` | `NrUeRrc` | TraceSources `MibReceived`, `Sib1Received`, `Sib2Received` | System information acquisition events are traced at UE RRC. |
| 13 | **RRC.Reconfig** | IMPLEMENTABLE | `model/nr-ue-rrc.cc`, `model/nr-gnb-rrc.cc` | `NrUeRrc`, `NrGnbRrc` | TraceSource `ConnectionReconfiguration` | RRC connection reconfiguration send/receive is traced on UE and gNB. |
| 14 | **UE count** (active RRC) | IMPLEMENTABLE | `model/nr-gnb-rrc.h` | `NrGnbRrc` | `GetUeMap().size()` | Returns the current number of UE contexts managed by gNB RRC. |

---

## E2 / O-RAN KPM (fork)

| # | Metric | Status | File(s) | Class | Function / TraceSource | Explanation |
|:-:|--------|:------:|---------|-------|----------------------|-------------|
| 1 | **DRB.UEThpDl** / KPM UE throughput | DERIVABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `BuildGUICuUp()` computes throughput; `BuildRicIndicationMessageCuUp()` reads `m_lastThroughputPerUe` | E2 uses throughput derived in GUI builder from PDCP byte deltas, not a native KPM counter. |
| 2 | **DRB.PdcpSduVolumeDl** (E2) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc`, `model/nr-rlc.h` | `NrGnbNetDevice`, `NrRlc` | `GetDlTxData()` read in builder; **encoded** via `AddPdcpUePmItem()` using RLC `GetTxBytesInReportingPeriod()` | PDCP volume is available from calculator; E2 message currently carries RLC byte counts under PDCP KPM names. |
| 3 | **DRB.PdcpSduDelayDl** (E2) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc`, `helper/nr-bearer-stats-calculator.cc` | `NrGnbNetDevice`, `NrBearerStatsCalculator` | `GetDlDelay()` aggregated in `BuildRicIndicationMessageCuUp()` | Per-UE PDCP delay is read from the E2 PDCP calculator during indication build. |
| 4 | **DRB.RlcSduVolumeDl** (E2) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc`, `model/nr-rlc.h` | `NrGnbNetDevice`, `NrRlc` | `GetTxPacketsInReportingPeriod()`, `GetTxBytesInReportingPeriod()` → `AddPdcpUePmItem()` | RLC PDU count and bytes over the E2 period are read from each DRB's RLC entity. |
| 5 | **RRC.ConnMean** (approx.) | DERIVABLE | `model/nr-gnb-net-device.cc`, `model/nr-gnb-rrc.h` | `NrGnbNetDevice`, `NrGnbRrc` | `m_rrc->GetUeMap().size()` → `AddPHYGnbConfiguration()` | E2 reports instantaneous connected-UE count; true time-averaged mean requires external aggregation. |
| 6 | Antenna ports on/off (CCC) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `GetPortPower()` → `AddPHYGnbConfiguration(numActiveUes, cellId, portsOn, portsOff)` | Port mask is counted into on/off tallies and sent in the KPM CU-UP indication. |
| 7 | E2 periodicity | IMPLEMENTABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | Attribute `E2Periodicity`; schedules `BuildAndSendReportMessage()` | Reporting interval is configured as a `NrGnbNetDevice` TypeId attribute (seconds). |
| 8 | CSV logging | IMPLEMENTABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `BuildGUICuUp()` → `m_cuUpFileName` (`nr-cu-up-cell-<id>.txt`); attribute `EnableE2FileLogging` | Periodic CSV mirrors E2/GUI metrics including throughput, ports, and CCC power fields. |

---

## Additional Fork Metrics (CCC power — not in original KPI tables)

These are available in the O-RAN fork and used by CCC/GUI but were not separate rows in the source doc:

| Metric | Status | File(s) | Class | Function | Explanation |
|--------|:------:|---------|-------|----------|-------------|
| CCC instantaneous gNB power (W) | IMPLEMENTABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `SampleTransmitPower()`, `GetAveragePower()` | Port- and PRB-aware power model runs every 100 ms after `DoInitialize()`. |
| Baseline vs xApp power comparison | DERIVABLE | `model/nr-gnb-net-device.cc` | `NrGnbNetDevice` | `SampleTransmitPower()` accumulators; `BuildGUICuUp()` CSV columns | Baseline/xApp min/max/accumulated power and savings % are derived from timed sampling windows. |

---

## Code Index (validated entry points)

| Component | Path |
|-----------|------|
| Trace enabler | `helper/nr-helper.cc` — `EnableTraces()` |
| PHY trace sink | `helper/nr-phy-rx-trace.h`, `helper/nr-phy-rx-trace.cc` |
| MAC trace sink | `helper/nr-mac-scheduling-stats.h`, `helper/nr-mac-scheduling-stats.cc` |
| RLC/PDCP stats | `helper/nr-bearer-stats-calculator.h`, `helper/nr-bearer-stats-calculator.cc` |
| PHY models | `model/nr-ue-phy.cc`, `model/nr-gnb-phy.cc`, `model/nr-spectrum-phy.cc` |
| MAC models | `model/nr-gnb-mac.cc`, `model/nr-ue-mac.cc` |
| RLC/PDCP | `model/nr-rlc.cc`, `model/nr-pdcp.cc` |
| RRC | `model/nr-gnb-rrc.cc`, `model/nr-ue-rrc.cc` |
| E2 / CCC KPM | `model/nr-gnb-net-device.cc` |
| UE power control | `model/nr-ue-power-control.cc` |

---

*Validated against 5G-LENA NR module in `ns-O-RAN-flexric/mmwave-LENA-oran`. No code was modified during this validation.*
