# RIC-TaaP Installation and Validation Manual

This manual describes how to set up the complete RIC-TaaP integration
environment on a fresh machine from the published repositories, and how to
validate that the installation is correct by running the **RF channel
reconfiguration target scenario** (KPM v3 observation → threshold/hysteresis
decision → dry-run `antenna-state` intent).

> **Validation boundary (important):** in the current FlexRIC checkout the RF
> xApp is a **KPM monitor with dry-run transport**. A successful run proves
> E2 registration, KPM subscription, indication handling, decision logic and
> clean teardown. It does **not** claim that a CCC command was encoded,
> transported or applied — native CCC reporting `native-ccc-unavailable` is
> the *expected* diagnostic, not an error. See
> [integration-report.md](integration-report.md) for the full evidence table.

---

## 1. Repositories and pinned revisions

| Component | Repository | Branch / ref | Expected commit |
|---|---|---|---|
| Top-level integration | `https://github.com/AndrzejDenisiewicz/ns-O-RAN-flexric.git` | `Integracja` | `af2dae4` |
| └ e2sim (E2 termination) | `https://github.com/MinaYonan123/e2sim-kpmv3.git` | submodule pin | `732d647` |
| └ ns-3 simulator (mmwave + 5G-LENA) | `https://github.com/AndrzejDenisiewicz/mmwave-LENA-oran.git` | `Integracja` | `40e7541` |
| &nbsp;&nbsp;&nbsp;└ 5G-LENA NR module | `https://github.com/AndrzejDenisiewicz/ns3-oran-lena-nr.git` | `Integracja` | `1c4f3b7` |
| &nbsp;&nbsp;&nbsp;└ ORAN interface module | `https://github.com/AndrzejDenisiewicz/oran-interface.git` | `Integracja` | `577c326` |
| Near-RT RIC + xApps | `https://github.com/AndrzejDenisiewicz/flexric.git` | `dev` | `9946e79` |

The parent repository pins the submodules to the exact commits above; a
recursive clone reproduces the validated tree. `flexric` is a sibling
checkout (not a submodule).

## 2. System requirements

- Ubuntu 22.04 LTS (recommended), ≥ 8 GB RAM, ≥ 20 GB free disk
- C/C++ toolchain with **gcc-13** (FlexRIC does not support gcc-11), CMake,
  SCTP headers, Autotools, Bison/Flex, Boost
- **Build order is significant: FlexRIC → e2sim → ns-3**

### 2.1 Install packages

```bash
sudo apt-get update

# e2sim + FlexRIC requirements
sudo apt-get install -y \
  build-essential git cmake cmake-curses-gui \
  gcc-13 g++-13 cpp-13 libc6-dev \
  libsctp-dev libpcre2-dev \
  autoconf automake libtool bison flex \
  libboost-all-dev

# ns-3 / 5G-LENA requirements
sudo apt-get install -y python3.8

# nlohmann JSON (required by the CCC service-model code)
sudo apt-get install -y nlohmann-json3-dev

# Make gcc-13 the default compiler (FlexRIC requirement)
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100 \
  --slave /usr/bin/g++ g++ /usr/bin/g++-13 \
  --slave /usr/bin/gcov gcov /usr/bin/gcov-13
sudo update-alternatives --config gcc        # choose gcc-13
gcc --version                                # must report 13.x
```

Optional (only if you need the corresponding features):

```bash
sudo apt-get install -y sqlite sqlite3 libsqlite3-dev   # LENA comparison examples
sudo apt-get install -y libeigen3-dev                   # MIMO/Eigen features
# Docker Compose (only for the RIC-TaaP Studio GUI): https://docs.docker.com/compose/install/
```

> **Ubuntu 24.04+:** do not install `python3.8` system-wide. Use a venv:
> `sudo add-apt-repository ppa:deadsnakes/ppa && sudo apt install python3.8 python3.8-venv python3.8-dev`
> then `python3.8 -m venv myenv && source myenv/bin/activate`.

## 3. Clone the repositories

```bash
export TOP=$HOME/ric-taap            # workspace root, adjust as needed
mkdir -p "$TOP" && cd "$TOP"

# 1) Top-level integration repo with all submodules
git clone https://github.com/AndrzejDenisiewicz/ns-O-RAN-flexric.git
cd ns-O-RAN-flexric
git checkout Integracja
git submodule update --init --recursive
cd "$TOP"

# 2) FlexRIC (sibling checkout)
git clone https://github.com/AndrzejDenisiewicz/flexric.git
cd flexric
git checkout dev
cd "$TOP"
```

### 3.1 Verify the tree matches the pinned revisions

```bash
cd "$TOP"
git -C ns-O-RAN-flexric rev-parse --short HEAD          # af2dae4
git -C ns-O-RAN-flexric/mmwave-LENA-oran rev-parse --short HEAD          # 40e7541
git -C ns-O-RAN-flexric/mmwave-LENA-oran/src/nr rev-parse --short HEAD          # 1c4f3b7
git -C ns-O-RAN-flexric/mmwave-LENA-oran/contrib/oran-interface rev-parse --short HEAD          # 577c326
git -C ns-O-RAN-flexric/e2sim-kpmv3 rev-parse --short HEAD          # 732d647
git -C flexric rev-parse --short HEAD          # 9946e79
```

If any value differs, re-run `git submodule update --init --recursive` inside
`ns-O-RAN-flexric` and `git checkout dev` inside `flexric`.

## 4. Build

### 4.1 FlexRIC (near-RT RIC + xApps)

Configure with **E2AP v1.01** and **KPM v3.00** (the versions used by the
simulator; the FlexRIC defaults differ):

```bash
cd "$TOP/flexric"
mkdir build && cd build
cmake .. -DE2AP_VERSION=E2AP_V1 -DKPM_VERSION=KPM_V3_00
make -j$(nproc)
sudo make install        # service models -> /usr/local/lib/flexric, config -> /usr/local/etc/flexric
```

This builds (among others):

- `build/examples/ric/nearRT-RIC` — the near-RT RIC
- `build/examples/xApp/c/orange/rf_reconfiguration_xapp` — the RF xApp (rebuild later with `cmake --build build --target rf_reconfiguration_xapp`)
- `build/examples/emulator/agent/emu_agent_gnb` — E2 agent emulator used by the smoke test

### 4.2 e2sim (E2 termination library)

Must be built **before ns-3** (the ORAN interface links against it):

```bash
cd "$TOP/ns-O-RAN-flexric/e2sim-kpmv3/e2sim"
./build_e2sim.sh 2
```

The argument is the log level (`2` = INFO, the documented default; `3` adds
ASN.1 xer-printing). The script builds, then removes and reinstalls the
`e2sim-dev` package (uses `sudo`).

### 4.3 ns-3 (mmwave + 5G-LENA NR + ORAN interface)

```bash
cd "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran"
./ns3 configure --disable-python --enable-modules='nr;oran-interface'
./ns3 build
```

Notes:

- `oran-interface` must be enabled together with `nr` so its public headers
  are generated before the NR integration compiles.
- This checkout's ns-3.42 core has no upstream `WraparoundModel`; CMake
  detects that and compiles `nr-wraparound-utils` with the documented
  fallback (a `WARNING: nr: ns-3 wraparound support unavailable; using tx
  mobility fallback` message is expected and harmless).
- `./ns3 configure && ./ns3 build` (full tree, with examples) also works but
  takes considerably longer.

### 4.4 (Optional) RIC-TaaP Studio GUI

```bash
cd "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran/GUI"
# edit docker-compose.yml: set NS3_HOST to the IP of the machine running ns-3
docker-compose up --build -d
pip3 install influxdb
```

Studio is served at `http://127.0.0.1:8000`, Grafana at
`http://127.0.0.1:3000` (admin/admin). First deployment can take up to 5
minutes. The terminal workflow below is fully valid without the GUI.

## 5. Installation verification

Run these checks before the scenario test. All of them must succeed.

```bash
# FlexRIC binaries
ls "$TOP/flexric/build/examples/ric/nearRT-RIC"
ls "$TOP/flexric/build/examples/xApp/c/orange/rf_reconfiguration_xapp"
ls "$TOP/flexric/build/examples/emulator/agent/emu_agent_gnb"

# Service models installed
ls /usr/local/etc/flexric/flexric.conf
ls /usr/local/lib/flexric | head

# e2sim installed
dpkg -l e2sim-dev | tail -1

# ns-3 modules built
ls "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran/build/lib/" | grep -E 'libnr|liboran-interface'

# Quick xApp sanity check: it must start cleanly and print its runtime
# configuration and "no E2 nodes found"-style diagnostics when no RIC is
# running (diagnostics, not a crash/segfault). Stop it manually afterwards.
"$TOP/flexric/build/examples/xApp/c/orange/rf_reconfiguration_xapp" &
XAPP_PID=$!; sleep 5; kill $XAPP_PID
```

## 6. Target scenario test — RF channel reconfiguration

### 6.1 Smoke test (RIC + E2 agent emulator, no ns-3 run)

This is the reproducible live validation described in
[xapp-rf-reconfiguration.md](xapp-rf-reconfiguration.md): `nearRT-RIC` +
`emu_agent_gnb` + an 8-second xApp observation window.

Open **three terminals**:

```bash
# Terminal 1 — near-RT RIC
cd "$TOP/flexric/build/examples/ric"
./nearRT-RIC
```

```bash
# Terminal 2 — E2 agent (registers one gNB E2 node with KPM/RC functions)
cd "$TOP/flexric/build/examples/emulator/agent"
./emu_agent_gnb
```

```bash
# Terminal 3 — RF reconfiguration xApp (default: kpm-first, dry-run)
cd "$TOP/flexric/build/examples/xApp/c/orange"
XAPP_DURATION=8 ./rf_reconfiguration_xapp
```

**Expected evidence (all must appear in the logs):**

| # | Where | Evidence |
|---|---|---|
| 1 | Terminal 1 (RIC) | E2 Setup Request/Response — **one E2 node registered** |
| 2 | Terminal 3 (xApp) | E2 node discovered; KPM RAN function ID `2` found |
| 3 | Terminal 3 (xApp) | KPM v3 report styles discovered and **subscribed** (both styles) |
| 4 | Terminal 3 (xApp) | KPM **indications received** (format 1 and format 3 real-valued measurements reach the decision callback) |
| 5 | Terminal 3 (xApp) | **`antenna-state` intent emitted** after threshold/hysteresis/cooldown logic (audit-only in dry-run) |
| 6 | Terminal 3 (xApp) | After `XAPP_DURATION` expires the xApp **deletes its subscriptions cleanly** and exits |

The agent reports contain random data (no UE is attached); the point of the
smoke test is the E2/KPM/decision/teardown chain, not KPI values.

### 6.2 Full loop with the ns-3 simulator

Same terminals 1 and 3; terminal 2 runs the RF reconfiguration scenario:

```bash
# Terminal 2 — ns-3 RF reconfiguration scenario (10 s)
cd "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran"
./ns3 run "scratch/RF_Reconfiguration.cc --simTime=10"
```

The scenario connects to the RIC at `127.0.0.1` by default (`e2TermIp`
global value; set `--e2TermIp=<ip>` for multi-host setups) and registers the
gNB with KPM function ID `2`, RC `3`, CCC `4`, E2 indication periodicity
`0.1 s` (`--indicationPeriodicity` to change). Start the RIC first, then the
scenario, then the xApp as in 6.1.

Expected: the same six evidence items as 6.1, with the E2 node provided by
the simulated gNB instead of the emulator, plus periodic KPM indications
driven by the simulated radio.

**Offline variant (no RIC required):**

```bash
./ns3 run "scratch/RF_Reconfiguration.cc --simTime=10 --enableE2FileLogging=true"
```

produces per-cell CU-UP KPI sample files (`cuup-cell-<id>.csv` by default)
instead of an E2 connection — useful to verify the simulator's KPI pipeline
alone.

### 6.3 Transport variants (xApp decision diagnostics)

Re-run the xApp (terminal 3) with each transport to confirm the documented
behavior. The observation path (KPM subscription, indications, audit log)
must be identical in all three; only the control diagnostic changes.

```bash
XAPP_DURATION=8 RF_XAPP_TRANSPORT=kpm-first         ./rf_reconfiguration_xapp   # default: no control sent (audit only)
XAPP_DURATION=8 RF_XAPP_TRANSPORT=native-ccc       ./rf_reconfiguration_xapp   # expected: "native-ccc-unavailable"
XAPP_DURATION=8 RF_XAPP_TRANSPORT=rc-compatibility ./rf_reconfiguration_xapp   # expected: typed intent retained, no CCC fidelity claim
```

| Variant | Control result | Status |
|---|---|---|
| `kpm-first` | No control sent | Recommended baseline |
| `native-ccc` | `native-ccc-unavailable` (no CCC encoder/API in this FlexRIC checkout) | Blocked — expected |
| `rc-compatibility` | Typed intent retained without CCC claim | Diagnostic only |

Other xApp knobs (defaults in parentheses): `RF_XAPP_POWER_THRESHOLD` (0),
`RF_XAPP_HYSTERESIS` (1), `RF_XAPP_COOLDOWN_MS` (5000), `RF_XAPP_DRY_RUN`
(true), `RF_XAPP_NODE_INDEX` (all nodes).

## 7. Acceptance criteria for the installation

1. §5 checks all pass.
2. Smoke test (§6.1) shows evidence items 1–6 in the logs.
3. Full-loop run (§6.2) completes normally and the same evidence appears with
   the simulator gNB as the E2 node.
4. Transport variants (§6.3) produce exactly the documented diagnostics —
   in particular, `native-ccc` must report **unavailable** (a different
   result means the wrong FlexRIC branch/build was used).
5. No component log contains crashes, SCTP connection failures or
   "no E2 nodes" diagnostics during the observation window.

## 8. Troubleshooting

| Symptom | Check |
|---|---|
| FlexRIC cmake fails / compile errors | `gcc --version` must be 13.x (gcc-11 unsupported). Re-run the `update-alternatives` steps in §2.1. |
| `e2sim` link errors during `./ns3 build` | e2sim must be built **before** ns-3 (§4.2). Re-run `./build_e2sim.sh 2`, then `./ns3 build`. |
| `nr` module fails on `wraparound-model.h` | Expected on the ns-3.42 core: the build must print the fallback WARNING and continue. If it errors instead, the `nr` submodule is not at `1c4f3b7` (§3.1). |
| xApp: "no E2 nodes found" | Is `nearRT-RIC` running and fully started before the agent/scenario/xApp? Is `NEAR_RIC_IP` correct (default `127.0.0.1`; override with `-a` or in `/usr/local/etc/flexric/flexric.conf`)? |
| E2 Setup never completes | E2AP runs over SCTP port **36421** — check it is not filtered by a firewall. Capture with Wireshark (`sctpport == 36421`) to inspect the setup exchange. |
| `native-ccc-unavailable` diagnostic | **Expected and correct** in this checkout — it is the documented validation boundary, not a fault. |
| Python errors during `./ns3` | Use Python 3.8 (system on 22.04, venv on 24.04+, §2.1 note). |
| Submodule content mismatch after clone/pull | `git submodule update --init --recursive` inside `ns-O-RAN-flexric`; re-verify commits with §3.1. |
| xApp exits immediately without logs | Run in the foreground without `XAPP_DURATION` to watch startup diagnostics; verify the xApp binary was built against the installed service models (`sudo make install` in FlexRIC). |

## 9. References

- [integration-report.md](integration-report.md) — full validation evidence and limitations
- [xapp-rf-reconfiguration.md](xapp-rf-reconfiguration.md) — RF xApp guide (dashboard, transport evaluation)
- [LENA_MODEL_REPORTING_PARAMETERS.md](LENA_MODEL_REPORTING_PARAMETERS.md) — 76-parameter KPI catalogue and mapping policies (`explicit-rejection`, `metadata-preserving`, `legacy-compatibility`; selected via the `KpmMappingPolicy` attribute on `NrGnbNetDevice`)
- [README.md](../README.md) — platform overview, GUI deployment, additional scenarios
- FlexRIC: `flexric/README.md` — RIC deployment and emulator details
