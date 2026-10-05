# RF Channel Reconfiguration xApp

## Overview

The RF Channel Reconfiguration xApp uses the **E2 Service Model (E2SM) Cell Configuration and Control (CCC)** O-RAN service model to dynamically adjust antenna port activation and transmission power at the simulated gNB. By reacting to observed network conditions, the xApp enables intelligent energy saving and performance optimization through RF-layer reconfiguration — without any manual intervention.

| Attribute | Value |
|-----------|-------|
| **Service Model** | CCC v06.00 (Cell Configuration and Control) |
| **Scenario File** | `scratch/RF_Reconfiguration.cc` |
| **xApp Binary** | `rf_reconfiguration_xapp` |
| **FlexRIC Branch** | `OIE-add-CCC` |
| **Reporting Used** | KPM v3.00 |

---

## How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                          near-RT RIC                             │
│                                                                  │
│  KPM CU-UP Indication ──► RF Reconfig xApp ──► CCC Control      │
│  (power, throughput,        (decision logic)    (port ON/OFF,    │
│   ports, energy)                                 power level)    │
└──────────▲──────────────────────────────────────────┬───────────┘
           │ E2 Interface                              │ E2 Interface
           │                                           ▼
┌────────────────────────────────────────────────────────────────┐
│                      ns-3 Simulation (5G-LENA)                 │
│                                                                │
│   NrGnbNetDevice ── CCC port/power model ── UEs               │
└────────────────────────────────────────────────────────────────┘
```

**Step-by-step operation:**

1. The xApp **subscribes** to KPM CU-UP indication messages from the gNB.
2. Each indication carries per-cell metrics including:
   - UE downlink throughput
   - Average and instantaneous transmit power
   - Active antenna port configuration (ports ON / OFF count)
   - Energy consumption samples
3. The xApp evaluates the metrics and sends a **CCC Control message** instructing the gNB to:
   - Increase or decrease the number of **active antenna ports**.
   - Adjust the **transmit power** scaling factor per port.
4. The gNB reconfigures its RF layer and reflects the new state in subsequent indication messages.
5. The closed loop repeats for the duration of the simulation.

---

## Dashboard

The **RF Channel Reconfiguration tab** in RIC-TaaP Studio provides a live view of the xApp operation:

![RF Channel Reconfiguration Dashboard](../fig/10.png)

| Panel | Description |
|-------|-------------|
| **UE DL Throughput** | Per-UE downlink throughput over time (Mbps) |
| **Network Average Power** | Average gNB transmit power (W) |
| **Indication Message Content** | Raw KPM indication fields received from the gNB |
| **Control Message Content** | CCC control fields sent by the xApp |
| **Power & Energy Consumption** | Per-cell power before and after reconfiguration |

---

## Prerequisites

Before running this xApp, make sure the following are in place:

- [ ] FlexRIC built from the `OIE-add-CCC` branch (see build instructions below).
- [ ] ns-O-RAN-flexric built (see [README §9.4](../README.md#94-ns-o-ran-flexric-installation)).
- [ ] RIC-TaaP Studio deployed (see [README §9.5](../README.md#95-gui-deployment)).

**Build FlexRIC with the CCC branch:**

```bash
cd /path/to/flexric
git checkout OIE-add-CCC
cd build
make -j8
sudo make install
```

---

## Running with RIC-TaaP Studio (Recommended)

**Step 1 — Start the near-RT RIC**

```bash
cd /path/to/flexric/build/examples/ric/
./nearRT-RIC
```

**Step 2 — Open RIC-TaaP Studio**

Open your browser at `http://127.0.0.1:8000`.

**Step 3 — Connect to FlexRIC**

Click **Connect to FlexRIC** on the homepage.

**Step 4 — Select and configure the scenario**

- Click **Show Form**.
- From the scenario list, select `RF_Reconfiguration.cc`.
- Click **Start**. The simulation will begin and cells/UEs will appear on the grid.

**Step 5 — Launch the RF Reconfiguration xApp**

```bash
cd /path/to/flexric/build/examples/xApp/c/orange/
./rf_reconfiguration_xapp
```

**Step 6 — Observe results**

Click the **RF Channel Reconfiguration** tab in RIC-TaaP Studio to monitor the live dashboard (see Dashboard section above).

**Step 7 — Stop the simulation**

Click **Stop** in the **Show Form** window.

---

## Running from Terminal

Open three separate terminals:

```bash
# Terminal 1 — near-RT RIC
cd /path/to/flexric/build/examples/ric/
./nearRT-RIC
```

```bash
# Terminal 2 — ns-3 simulation
cd /path/to/mmwave-LENA-oran
./ns3 run scratch/RF_Reconfiguration.cc
```

```bash
# Terminal 3 — RF Reconfiguration xApp
cd /path/to/flexric/build/examples/xApp/c/orange/
./rf_reconfiguration_xapp
```

---

## CCC Service Model — Technical Details

The CCC v06.00 service model was implemented from scratch as part of this project. The key components in the 5G-LENA layer are:

| Component | File | Function |
|-----------|------|----------|
| Port power control | `model/nr-gnb-net-device.cc` | `GetPortPower()`, `SetPortPower()`, `GetPortPowerScaling()` |
| Port state reporting | `model/nr-gnb-net-device.cc` | `GetPortPower()` → `AddPHYGnbConfiguration(numActiveUes, cellId, portsOn, portsOff)` |
| Power sampling | `model/nr-gnb-net-device.cc` | `SampleTransmitPower()` — runs every 100 ms after `DoInitialize()` |
| E2 reporting period | `model/nr-gnb-net-device.cc` | Attribute `E2Periodicity` on `NrGnbNetDevice` (seconds) |
| CSV logging | `model/nr-gnb-net-device.cc` | `BuildGUICuUp()` → `nr-cu-up-cell-<id>.txt`; attribute `EnableE2FileLogging` |

**Power model details:**

| Metric | Description |
|--------|-------------|
| Instantaneous power | Computed from PRB utilization and antenna activity model |
| Baseline power | Sampled before xApp execution for comparison |
| xApp power | Sampled after reconfiguration |
| Savings (%) | `(baseline − xApp) / baseline × 100` derived from sampling windows |

---

## Related Documentation

- [README — Platform Overview](../README.md)
- [Energy Saving xApp](xapp-energy-saving.md)
- [LENA Module Reporting Parameters](LENA_MODEL_REPORTING_PARAMETERS.md)

## Validation Status

The completed validation builds the simulator and runs identical one-cell,
one-UE offline cases for `explicit-rejection`, `metadata-preserving`, and
`legacy-compatibility`. With FlexRIC's `emu_agent_gnb`, the live smoke test
registered one E2 node, completed E42 setup, subscribed to both KPM report
styles, received indications, emitted an antenna-state intent, and deleted
both subscriptions cleanly. The run used dry-run mode, so no CCC antenna-mask
application or RF state change is claimed.

## `rf_reconfiguration_xapp`

The shared observation and decision executable is built from the FlexRIC
checkout with:

```sh
cmake --build build --target rf_reconfiguration_xapp
```

It discovers KPM report styles, subscribes to each selected E2 node, logs
normalized indications, and emits a typed antenna-state intent after applying
hysteresis and cooldown. The default mode is `kpm-first` and dry-run; no
control protocol is claimed or sent by default.

The decision engine can be configured without changing scenario inputs:

- `RF_XAPP_POWER_THRESHOLD` — decision threshold, default `0`.
- `RF_XAPP_HYSTERESIS` — dead-band around the threshold, default `1`.
- `RF_XAPP_COOLDOWN_MS` — minimum time between state changes, default `5000`.
- `RF_XAPP_DRY_RUN` — keep intents audit-only, default `true`.
- `RF_XAPP_NODE_INDEX` — restrict subscriptions to one discovered node index.
- `RF_XAPP_TRANSPORT` — `kpm-first`, `native-ccc`, or `rc-compatibility`.

`native-ccc` currently reports `native-ccc-unavailable` because this FlexRIC
checkout has no CCC encoder/API. `rc-compatibility` reports that the typed
intent was retained without claiming CCC fidelity; these diagnostics are the
baseline for the transport-adapter evaluation.

### Transport evaluation

The same executable was built once and exercised with each transport setting:

| Variant | Observation path | Control result | Status |
| --- | --- | --- | --- |
| `kpm-first` | KPM subscription and audit log | No control sent | Recommended milestone |
| `native-ccc` | KPM subscription and audit log | Explicit `native-ccc-unavailable` diagnostic | Blocked by missing FlexRIC CCC API |
| `rc-compatibility` | KPM subscription and audit log | Typed intent retained; no CCC claim | Adapter still required |

The live evidence used the same target and scenario-independent configuration
with `nearRT-RIC`, `emu_agent_gnb`, and an eight-second xApp observation window.
The xApp was stopped by the planned timeout after clean subscription teardown;
the registered node and received KPM indications were confirmed in the RIC,
agent, and xApp logs. The build result is reproducible with `UNIT_TEST=FALSE`
and the target `rf_reconfiguration_xapp`.

`kpm-first` is selected for the next validation milestone because it provides
the complete observation, threshold, hysteresis, cooldown, node-filter, and
audit path without misrepresenting either RC compatibility or native CCC
interoperability. Native CCC remains preferred only if a future FlexRIC
service-model/API addition is implemented and reaches the simulator callback.
