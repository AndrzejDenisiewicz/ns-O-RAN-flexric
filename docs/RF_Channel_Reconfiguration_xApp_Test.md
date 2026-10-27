# RF Channel Reconfiguration xApp — Test Manual

**Scope:** end-to-end test of the RF Channel Reconfiguration xApp (O-RAN Use Case 21,
sub-use case 4.21.3.2) on the RIC-TaaP platform, using the **AndrzejDenisiewicz**
GitHub repositories, branch/revisions of the `Integracja2` integration.

**What the test proves:** the xApp observes KPM indications from a simulated gNB and,
via the **native E2SM-CCC v06.00** path (not dry-run), sends an antenna-mask control
message that the simulator applies to the gNB's RF layer — transmit-port power
scaling changes from `1` to `0.5` after mask `1100` is accepted on a 4-port antenna.

```
┌─────────────┐  E42 (SCTP :36422)  ┌──────────────┐  E2AP (SCTP :36421)  ┌──────────────────┐
│  xApp       │ ◄─────────────────► │  nearRT-RIC  │ ◄───────────────────► │  ns-3 simulator  │
│  xapp_rf_   │   setup / control / │  (FlexRIC)   │   E2 Setup / KPM /   │  RF_Reconfiguration│
│  reconfig   │   KPM indications   │  + ccc_sm    │   CCC control        │  .cc (5G-LENA)    │
└─────────────┘                     └──────────────┘                      └──────────────────┘
```

> This manual records a run that **passed** on Ubuntu 24.04 (gcc 13.3, cmake 3.28),
> including two source-level fixes that are mandatory with the pinned revisions
> (Section 4 and Section 5.1). Evidence checklist: Section 9.

---

## 1. Repositories and pinned revisions

All from GitHub user **AndrzejDenisiewicz**. Clone order matters (submodules first).

| # | Component | Repository | Ref | Expected commit |
|---|-----------|-----------|-----|-----------------|
| 1 | Top-level integration | `https://github.com/AndrzejDenisiewicz/ns-O-RAN-flexric.git` | branch `Integracja2` | `85d7355` |
| 2 | E2 termination (e2sim) | submodule `e2sim-kpmv3` → `https://github.com/AndrzejDenisiewicz/e2sim-kpmv3.git` | submodule pin (branch `ccc-integracja`) | `223a155` |
| 3 | ns-3 simulator (mmWave + 5G-LENA) | submodule `mmwave-LENA-oran` → `https://github.com/AndrzejDenisiewicz/mmwave-LENA-oran.git` | submodule pin (branch `ccc-integracja`) | `718b4a2` |
| 4 | 5G-LENA NR module | nested submodule `mmwave-LENA-oran/src/nr` → `https://github.com/AndrzejDenisiewicz/ns3-oran-lena-nr.git` | branch `ccc-integracja` | `3881d30c` |
| 5 | ORAN interface module | nested submodule `mmwave-LENA-oran/contrib/oran-interface` → `https://github.com/AndrzejDenisiewicz/oran-interface.git` | branch `ccc-integracja` | `52d1224` |
| 6 | Near-RT RIC + xApps (FlexRIC) | `https://github.com/AndrzejDenisiewicz/flexric.git` | branch `ccc-integracja` | `cd458f36` (descendant of the CCC baseline `fe338ade`) |

Sanity checks after cloning:

```bash
test -d  $TOP/flexric/src/sm/ccc_sm                                  # CCC service model
test -f  $TOP/flexric/examples/xApp/c/orange/rf_reconfiguration_xapp.c
git -C $TOP/ns-O-RAN-flexric/e2sim-kpmv3 rev-parse --short HEAD      # 223a155
git -C $TOP/ns-O-RAN-flexric/mmwave-LENA-oran rev-parse --short HEAD # 718b4a2
```

**Nested-submodule caveat (known):** the gitlink commits pinned by
`mmwave-LENA-oran@718b4a2` for `src/nr` and `contrib/oran-interface`
(`a3c9096`, `43f5fd1`) were lost on the remotes (force-push), so
`git submodule update --init --recursive` fails with `not our ref`.
The live, CCC-capable equivalents are the **`ccc-integracja` branch tips** —
check those out (Section 3, step 3). They carry the committed versions of the
CCC callback / KPM queue / CccFunctionDescription changes this test needs.

The same applies to `e2sim-kpmv3/e2sim/e2sm_examples/kpm_e2sm/nlohmann_json_release`
(pinned commit `4f8fba1` is gone). It is only referenced by the e2sim Dockerfile,
**not** by the CMake build — ignore it, or vendor `json.hpp` from
`https://github.com/azadkuh/nlohmann_json_release.git` (the repo the Dockerfile uses).
FlexRIC's `ccc_sm` finds/downloads nlohmann/json on its own (FetchContent fallback),
so no system `nlohmann-json3-dev` package is required.

---

## 2. System requirements

- Ubuntu 22.04/24.04, ≥ 8 GB RAM, ≥ 20 GB disk (build tree ~10 GB)
- **gcc-13 / g++-13** (FlexRIC does not support gcc-11; Ubuntu 24.04 default is 13.3 ✓)
- cmake ≥ 3.15, git, C/C++ toolchain
- Kernel **SCTP** available (the e2sim fork uses kernel SCTP sockets
  `socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP)` — no userspace `libsctp` in the
  simulator data path; FlexRIC/xApp binaries do link `-lsctp`):
  ```bash
  lsmod | grep sctp        # expect: sctp (+ sctp_diag)
  # or test: python3 -c "import socket; socket.socket(2,5,132).close(); print('ok')"
  ```
- Packages:
  ```bash
  sudo apt-get install -y build-essential git cmake autoconf automake libtool \
      bison flex libsctp-dev libpcre2-dev libboost-all-dev python3
  # Ubuntu 22.04 only: gcc-13 + make it default
  #   sudo apt-get install -y gcc-13 g++-13
  #   sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-13 100 \
  #       --slave /usr/bin/g++ g++ /usr/bin/g++-13
  ```

Build order is significant: **FlexRIC → e2sim → ns-3**.

---

## 3. Clone the repositories

```bash
export TOP=/home/adminradio/Projekty/Andrzej/Integracja_Test   # adjust

# 1) Top-level integration repo + submodules
git clone --branch Integracja2 https://github.com/AndrzejDenisiewicz/ns-O-RAN-flexric.git "$TOP"
cd "$TOP/ns-O-RAN-flexric"
git submodule update --init e2sim-kpmv3 mmwave-LENA-oran        # NOT --recursive (see caveat)

# 2) Nested modules: the pinned gitlinks are lost on the remotes;
#    use the live ccc-integracja branch tips instead:
git -C mmwave-LENA-oran/contrib/oran-interface fetch origin '+refs/heads/*:refs/remotes/origin/*'
git -C mmwave-LENA-oran/contrib/oran-interface checkout -B ccc-integracja origin/ccc-integracja
git -C mmwave-LENA-oran/src/nr fetch origin '+refs/heads/*:refs/remotes/origin/*'
git -C mmwave-LENA-oran/src/nr checkout -B ccc-integracja origin/ccc-integracja

# 3) FlexRIC (sibling checkout, CCC-capable branch)
git clone --branch ccc-integracja https://github.com/AndrzejDenisiewicz/flexric.git "$TOP/flexric"
test -d "$TOP/flexric/src/sm/ccc_sm"
```

---

## 4. Mandatory source fix — e2sim SCTP client bind (IPv4)

The pinned e2sim commit `223a155` ("SCTP client socket uses configured address
family") fixed the `socket()` call to use the server's family, but left the
client `bind()` hardcoded to IPv6. Connecting to an IPv4 RIC (`127.0.0.1`) then
fails with `bind: EINVAL` and the simulator aborts (`NS_ASSERT ... ptr.h`) before
any E2 Setup is exchanged.

Apply this patch to `ns-O-RAN-flexric/e2sim-kpmv3/e2sim/src/SCTP/e2sim_sctp.cpp`
(function `sctp_start_client`, ~line 162):

```diff
   if (setsockopt(client_fd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof optval) != 0) {
       LOG_E("setsockopt addr, error message: %s", strerror(errno));
       exit(EXIT_FAILURE);
   }
 
-    struct sockaddr_in6  client6_addr {};
-    client6_addr.sin6_family = AF_INET6;
-    client6_addr.sin6_port   = htons(client_port);
-    client6_addr.sin6_addr   = in6addr_any;
-
   LOG_I("[SCTP] Binding client socket with source port %d", client_port);
-  if(bind(client_fd,(struct sockaddr*) &client6_addr,sizeof(client6_addr)) == -1) {
-    LOG_E("bind");
-    exit(EXIT_FAILURE);
-  }
+  if (server_addr->sa_family == AF_INET) {
+    struct sockaddr_in client4_addr {};
+    client4_addr.sin_family = AF_INET;
+    client4_addr.sin_port   = htons(client_port);
+    client4_addr.sin_addr.s_addr = htonl(INADDR_ANY);
+
+    if(bind(client_fd,(struct sockaddr*) &client4_addr,sizeof(client4_addr)) == -1) {
+      LOG_E("bind");
+      exit(EXIT_FAILURE);
+    }
+  }
+  else {
+    struct sockaddr_in6  client6_addr {};
+    client6_addr.sin6_family = AF_INET6;
+    client6_addr.sin6_port   = htons(client_port);
+    client6_addr.sin6_addr   = in6addr_any;
+
+    if(bind(client_fd,(struct sockaddr*) &client6_addr,sizeof(client6_addr)) == -1) {
+      LOG_E("bind");
+      exit(EXIT_FAILURE);
+    }
+  }
   // end binding ---------------------
```

Expected symptom without the fix (simulator log, then abort):

```
[INFO ] [SCTP] Binding client socket with source port 0
[ERROR] bind
NS_ASSERT failed, cond="m_ptr", msg="Attempted to dereference zero pointer" ... src/core/model/ptr.h, line=718
```

---

## 5. Build

### 5.1 FlexRIC (nearRT-RIC + xApps) — **must be a Release build**

Configure, build, install (service models + config):

```bash
cd "$TOP/flexric"
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release \
         -DE2AP_VERSION=E2AP_V1 \
         -DKPM_VERSION=KPM_V3_00 \
         -DUNIT_TEST=FALSE
make -j"$(nproc)"
sudo make install        # SMs -> /usr/local/lib/flexric, conf -> /usr/local/etc/flexric
```

**Why Release is mandatory:** the simulator creates **two E2 terminations per gNB**
(known integration caveat — the scenario logs `Start E2 Agent (E2 Simulator)` twice),
so the RIC receives a second `E2 Setup Request` for the **same** Global RAN Node ID.
In a default (asserts-on) build this hits

```
src/lib/msg_hand/reg_e2_nodes.c:114: add_reg_e2_node_v1:
Assertion `it_node == end_node && "Trying to add an already existing E2 Node"' failed.
```

and the RIC aborts — which also drops the xApp's E42 connection (the xApp then
spams `[E2AP]: Resending Setup Request after timeout`) and loses the first E2
association. With `CMAKE_BUILD_TYPE=Release` (`-DNDEBUG`) the assert is compiled
out and the RIC tolerates the duplicate, exactly like the validated reference run.

Verify the installed pieces:

```bash
ls /usr/local/lib/flexric | grep -E 'ccc_sm|kpm_sm|rc_sm'   # libccc_sm.so must be present
ls /usr/local/etc/flexric/flexric.conf
```

Key binaries (run from the build tree):

```
$TOP/flexric/build/examples/ric/nearRT-RIC
$TOP/flexric/build/examples/xApp/c/orange/xapp_rf_reconfiguration   # built from rf_reconfiguration_xapp.c
$TOP/flexric/build/examples/emulator/agent/emu_agent_gnb            # optional agent emulator
```

> **No-sudo alternative:** add `-DCMAKE_INSTALL_PREFIX="$TOP/prefix"` to `cmake`
> (drop `sudo`). The SMs then land in `$TOP/prefix/lib/flexric` and the config in
> `$TOP/prefix/etc/flexric/flexric.conf`; start every FlexRIC binary with
> `-c "$TOP/prefix/etc/flexric/flexric.conf" -p "$TOP/prefix/lib/flexric/"`.

### 5.2 e2sim (E2 termination library)

```bash
cd "$TOP/ns-O-RAN-flexric/e2sim-kpmv3/e2sim"
mkdir build && cd build
cmake .. -DDEV_PKG=1 -DLOG_LEVEL=2     # LOG_LEVEL 2=INFO (3 adds ASN.1 xer print)
make -j"$(nproc)"
sudo make install                      # headers -> /usr/local/include/e2sim, libe2sim.a -> /usr/local/lib
```

(With no sudo use `-DCMAKE_INSTALL_PREFIX="$TOP/prefix"` and, for the ns-3 build
in 5.3, pass the matching `-I/-L` flags. Do **not** copy the bare
`libe2sim_shared.so` from this build into the prefix/lib link path — it lacks the
ASN.1 objects (they live only in `libe2sim.a`) and breaks the module link with
`undefined reference to asn_DEF_*`.)

### 5.3 ns-3 (mmWave + 5G-LENA NR + ORAN interface)

```bash
cd "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran"
./ns3 configure --disable-python --enable-modules='energy;mmwave;nr;oran-interface'
cmake --build cmake-cache --target ns3.42-RF_Reconfiguration-default -j"$(nproc)"
```

Notes:

- **Module set.** The repo's installation manual lists `--enable-modules='nr;oran-interface'`,
  but `scratch/RF_Reconfiguration.cc` references `ns3::MmWaveHelper` E2 attributes, so the
  **`mmwave`** module must be built (otherwise the scenario dies at startup with
  `Could not set default value for ns3::MmWaveHelper::E2TermIp`), and `mmwave` needs
  **`energy`**. Use `'energy;mmwave;nr;oran-interface'`.
- **Do not run a full `./ns3 build`**: the unrelated stale scenario
  `scratch/orange-rf-channel-reconfiguration.cc` calls the removed API
  `NrHelper::ActivateDedicatedEpsBearer` (renamed `ActivateDedicatedQosFlow` in this
  NR module revision) and fails the build. Build only the needed target, as above.
  (Expected harmless warning during `nr` compile: ns-3.42 core has no
  `WraparoundModel` → `nr-wraparound-utils` fallback.)
- **No-sudo variant** (e2sim installed under `$TOP/prefix`):
  ```bash
  P="$TOP/prefix"
  ./ns3 configure --disable-python --enable-modules='energy;mmwave;nr;oran-interface' -- \
      -DCMAKE_C_FLAGS="-I$P/include -I$P/include/e2sim" \
      -DCMAKE_CXX_FLAGS="-I$P/include -I$P/include/e2sim" \
      "-DCMAKE_EXE_LINKER_FLAGS=-L$P/lib -lpthread" \
      "-DCMAKE_SHARED_LINKER_FLAGS=-L$P/lib -lpthread -Wl,-rpath,$P/lib"
  ```
  (The module CMakeLists hardcode `/usr/local/include/e2sim` and
  `link_directories(/usr/local/lib)`; the flags above add the prefix paths. No
  `-lsctp` is needed for the simulator side — kernel SCTP only.)
- If you later replace the e2sim library, ninja does **not** re-link the scenario
  (it doesn't track libs linked via `-L` flags):
  `rm build/scratch/ns3.42-RF_Reconfiguration-default` and rebuild the target.

---

## 6. Pre-flight checks

```bash
# Binaries
ls "$TOP/flexric/build/examples/ric/nearRT-RIC"
ls "$TOP/flexric/build/examples/xApp/c/orange/xapp_rf_reconfiguration"
ls "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran/build/scratch/ns3.42-RF_Reconfiguration-default"

# No stale processes holding the E2/E42 ports (common after earlier test sessions!)
ss -ln | grep -E '36421|36422'
pgrep -af 'nearRT-RIC|xapp_rf_reconfiguration'
# kill stale ones (they are just idle test processes):
#   kill <pids>   (SIGTERM; SIGKILL if stuck in a blocking E42/SCTP call)
```

Ports: E2AP = SCTP **36421** (RIC ↔ simulator), E42 = SCTP **36422** (RIC ↔ xApp).

---

## 7. Run the test

Open three terminals. **Start order: RIC → simulator → xApp** (the xApp sleeps 4 s
before discovering E2 nodes; the simulator must be registered by then).

```bash
# Terminal 1 — near-RT RIC
cd "$TOP/flexric/build/examples/ric"
./nearRT-RIC
#   (no-sudo: ./nearRT-RIC -c "$TOP/prefix/etc/flexric/flexric.conf" -p "$TOP/prefix/lib/flexric/")
```

```bash
# Terminal 2 — ns-3 RF reconfiguration scenario (10 s of simulation time,
# CCC RAN function ID 4, E2 indication periodicity 0.1 s)
cd "$TOP/ns-O-RAN-flexric/mmwave-LENA-oran"
./build/scratch/ns3.42-RF_Reconfiguration-default --CCC_E2functionID=4 --simTime=10
```

```bash
# Terminal 3 — RF Reconfiguration xApp, native CCC transport, LIVE control (no dry-run)
cd "$TOP/flexric/build/examples/xApp/c/orange"
RF_XAPP_TRANSPORT=native-ccc RF_XAPP_DRY_RUN=0 ./xapp_rf_reconfiguration
#   (no-sudo: add -c "$TOP/prefix/etc/flexric/flexric.conf" -p "$TOP/prefix/lib/flexric/")
```

Run sequence to expect:

1. RIC listens on 36421.
2. Simulator starts: 1 gNB / 1 UE, `CCC SetPortPower applied on cell 1: scaling=1`
   (baseline), E2 Setup → `SETUP-RESPONSE-SUCCESS`. (A second `Start E2 Agent`
   appears ~2 s later — the known double-E2Termination; the RIC tolerates it in a
   Release build.)
3. xApp: E42 setup, discovers **2** E2 nodes (one per termination), sends the
   deterministic smoke control `antennaMask=1100` ~4 s after start, receives the
   CONTROL ACK, subscribes to KPM (RAN function ID 2) on both nodes.
4. Simulator applies the mask: power scaling `1 → 0.5`, sends E2-SERVICE-UPDATE.
5. Simulation runs 10 s (≈ 1 minute wall time) with KPM indications every 0.1 s,
   then prints the performance summary and exits.

---

## 8. xApp runtime configuration (env vars)

| Variable | Default | Meaning |
|---|---|---|
| `RF_XAPP_TRANSPORT` | `kpm-first` | `kpm-first` (audit only, **no control sent**), `native-ccc` (sends CCC control via `SM_CCC_ID`), `rc-compatibility` (typed intent, no CCC claim) |
| `RF_XAPP_DRY_RUN` | `true` | `1` = intents are audit-only. For a live CCC run set **`RF_XAPP_DRY_RUN=0`** *and* `RF_XAPP_TRANSPORT=native-ccc` |
| `RF_XAPP_POWER_THRESHOLD` | `0` | decision threshold for KPI-based antenna-state changes |
| `RF_XAPP_HYSTERESIS` | `1` | dead-band around the threshold |
| `RF_XAPP_COOLDOWN_MS` | `5000` | minimum time between state changes |
| `RF_XAPP_NODE_INDEX` | all nodes | restrict subscriptions to one discovered node index |

In `native-ccc` + live mode the xApp sends one deterministic `1100` smoke control
right after E2 setup and a second one ~9 s later; the second is deduplicated
(`CCC control already sent; suppressing duplicate request`) unless the KPI decision
logic emits a different state.

---

## 9. Expected evidence (pass/fail checklist)

All of the following must appear in the three logs.

**Terminal 1 — nearRT-RIC:**

```
[NEAR-RIC]: Loading SM ID = 4 with def = ORAN-E2SM-CCC
[E2AP]: E2 SETUP-REQUEST rx from PLMN ... RAN type ngran_gNB
[NEAR-RIC]: Accepting RAN function ID 4 with def = ORAN-E2SM-CCC
[NEAR-RIC]: Accepting RAN function ID 3 with def = ORAN-E2SM-RC
[NEAR-RIC]: Accepting RAN function ID 2 with def = ORAN-E2SM-KPM
[iApp]: E42_RIC_CONTROL_REQUEST rx
[NEAR-RIC]: CONTROL SERVICE sent
[iApp]: SUBSCRIPTION-REQUEST RAN_FUNC_ID 2 ... tx
```

**Terminal 2 — simulator:**

```
CCC SetPortPower applied on cell 1: scaling=1                 # baseline
[INFO ] [SCTP] Sent E2-SETUP-REQUEST
[INFO ] [E2AP] Received SETUP-RESPONSE-SUCCESS
[INFO ] %%we are getting the e2sm callback for func id 4      # CCC control arrived
[INFO ] RIC control callback lookup succeeded for ranFunction=4
CCC callback received for cell 1
CCC decoding control PDU for cell 1
CCC control PDU decoded for cell 1
CCC antennaMask accepted for cell 1 (4 ports)                 # mask 1100 validated
CCC SetPortPower applied on cell 1: scaling=0.5               # *** THE PASS CRITERION ***
[INFO ] [SCTP] Sent E2-SERVICE-UPDATE
=== NR MIMO SIMULATION COMPLETE ===
```

**Terminal 3 — xApp:**

```
[xApp]: RF transport=native-ccc, dry_run=0
[CCC SM Agent]: Initialized with JSON-based encoding
[xApp]: E42 SETUP-RESPONSE rx
[KPM CCC]: Connected E2 nodes = 2
[xApp]: Sending native CCC smoke control after E2 setup
[xApp]: Sending RF Reconfiguration Control message - antenna mask: 1100 (old: 0000)
[xApp]: CONTROL-REQUEST tx
[xApp]: CONTROL ACK rx
[xApp]: Successfully received CONTROL response
[xApp]: E42 RIC SUBSCRIPTION REQUEST tx RAN_FUNC_ID 2 ...
[xApp]: Successfully subscribed to RAN_FUNC_ID 2
[xApp]: CCC control already sent; suppressing duplicate request
```

### Acceptance criteria

1. E2 Setup completes for the simulated gNB (KPM `2`, RC `3`, CCC `4` all accepted).
2. xApp receives a **CONTROL ACK** for the `1100` antenna-mask control (no dry-run).
3. Simulator logs `CCC antennaMask accepted for cell 1 (4 ports)` and
   **`CCC SetPortPower applied ... scaling` changes from `1` to `0.5`**.
4. KPM subscription succeeds on both E2 nodes.
5. No crashes, no `bind` errors, no `native-ccc-unavailable` diagnostic, and no
   `Trying to add an already existing E2 Node` abort in any log.
6. Simulation completes and prints the performance summary
   (healthy run: ~250+ Mbps single-flow DL, mean delay a few ms).

---

## 10. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `bind: EINVAL` + `NS_ASSERT ... ptr.h` in simulator at startup | e2sim IPv4/IPv6 bind bug — apply the Section 4 patch, rebuild e2sim, re-link the scenario (Section 5.3 note) |
| RIC: `add_reg_e2_node_v1: Assertion "Trying to add an already existing E2 Node"` | Default (asserts-on) FlexRIC build. Rebuild with `-DCMAKE_BUILD_TYPE=Release` (Section 5.1) |
| xApp spams `[E2AP]: Resending Setup Request after timeout` | RIC not running / just died (check Terminal 1), or stale RIC from another session holding port 36421/36422 — `ss -ln \| grep 3642` and kill stragglers |
| `Could not set default value for ns3::MmWaveHelper::E2TermIp` (NS_FATAL) | `mmwave` module not built — add it to `--enable-modules` (and `energy`), rebuild |
| Link errors `undefined reference to asn_DEF_*` / `xer_decode` | A partial `libe2sim_shared.so` (without ASN.1 objects) was picked up instead of `libe2sim.a` — remove the stray `.so` from the link path; link the static archive (Section 5.2 note) |
| `ninja: no work to do` after replacing e2sim lib | Ninja doesn't track `-L`-linked prefix libs — `rm build/scratch/ns3.42-RF_Reconfiguration-default` and rebuild the target |
| Full `./ns3 build` fails on `orange-rf-channel-reconfiguration.cc` (`ActivateDedicatedEpsBearer`) | Stale unrelated scenario vs. renamed NR API — build only `ns3.42-RF_Reconfiguration-default` (Section 5.3) |
| `xApp: no E2 nodes` / `Connected E2 nodes = 0` | RIC started too late or `NEAR_RIC_IP` in flexric.conf wrong (default `127.0.0.1`); start RIC first, fully, before the others |
| `native-ccc-unavailable` from the xApp | Wrong FlexRIC branch/build — must be a CCC-capable revision containing `src/sm/ccc_sm` (branch `ccc-integracja`, `libccc_sm.so` installed) |
| FlexRIC cmake/compile errors about gcc | gcc must be 13.x (`gcc --version`); on Ubuntu 22.04 set up `gcc-13` alternatives |
| xApp log appears empty while running | stdout is fully buffered when redirected to a file; lines appear on flush/exit. Run in a terminal or use `stdbuf -oL` |
| Two `Start E2 Agent (E2 Simulator)` blocks in the simulator log | Known caveat — two E2Terminations per gNB; harmless (control is sent twice, xApp deduplicates; RIC Release build tolerates the duplicate registration) |
| E2 never establishes, firewall suspected | E2AP = SCTP port 36421 (Wireshark filter `sctpport == 36421`) |

---

## 11. Cleanup

```bash
pkill -TERM -f nearRT-RIC        # or kill <pid>  (xApp, scenario if still running)
rm -f gnbs.txt ue_position.txt   # simulator working files in the mmwave-LENA-oran dir
# optional: remove run artifacts (default/, *.txt KPI/trace files) created by the run
```

Do not commit simulator outputs or build trees; keep the e2sim patch (Section 4)
as a local working-tree change until it is merged upstream.

---

## 12. References

- `ns-O-RAN-flexric/docs/xapp-rf-reconfiguration.md` — xApp concept, dashboard, transports
- `ns-O-RAN-flexric/docs/integration-report.md` — validation evidence table
- `ns-O-RAN-flexric/docs/installation-manual.md` — full platform install (incl. GUI/Docker)
- `flexric/examples/xApp/c/orange/rf_reconfiguration_xapp.c` — xApp source (transport,
  hysteresis/cooldown, deterministic `1100` smoke control, dedup)
- `mmwave-LENA-oran/scratch/RF_Reconfiguration.cc` — scenario (1 cell / 1 UE, traffic
  pattern, `--CCC_E2functionID`, E2 Termination, port-power baseline)
- `e2sim-kpmv3/e2sim/src/CCC-SM/e2sm_ccc.hpp` — CCC JSON encode/decode (nlohmann)
- `src/nr/model/nr-gnb-net-device.cc` (nr module) — `SetPortPower`, `SampleTransmitPower`,
  CCC callback, thread-safe KPM pending queue
