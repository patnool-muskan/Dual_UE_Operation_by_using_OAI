# OAI 5G Multi-UE Network Slicing Testbed (eMBB + URLLC)

A 3-PC, 3-USRP OpenAirInterface (OAI) 5G testbed running a single Core + gNB instance that serves **two UEs on two separate network slices** — eMBB and URLLC — verified end-to-end with iperf3.

## Contents

- [Result](#result)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [Repo structure](#repo-structure)
- [Network slice configuration](#network-slice-configuration)
- [Step-by-Step Setup](#step-by-step-setup)
- [Testing throughput per slice](#testing-throughput-per-slice)
- [Troubleshooting log](#troubleshooting-log)
- [Known notes / lab-only credentials](#known-notes--lab-only-credentials)
- [License](#license)

## Result

✅ Multi-UE, multi-slice operation confirmed working. Both UEs attached, established PDU sessions on their respective slices, and exchanged UDP traffic with the core's external Data Network container at 2 Mbit/s with **0% packet loss** over a 30-second test.

## Quick Start

> Assumes OAI CN5G, OAI RAN (gNB + nrUE), and UHD drivers are already built on all three PCs, and all three USRPs are connected and RF-synced. For a from-scratch setup, see [Step-by-Step Setup](#step-by-step-setup) below.

```bash
# 1. On the Core PC — bring up the 5G core
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
docker compose up -d
docker compose ps                       # all NFs + mysql + ims + ext-dn should be "healthy"

# 2. On the Core PC — start the gNB
sudo ./nr-softmodem -O <path-to-gnb-conf> --sa -E --continuous-tx

# 3. On UE1's PC (eMBB) — start the UE with the eMBB slice config
sudo ./nr-uesoftmodem -O ue_embb.conf --sa -r 106 --numerology 1 -E --continuous-tx --uicc0.imsi 001010000000001

# 4. On UE2's PC (URLLC) — start the UE with the URLLC slice config
sudo ./nr-uesoftmodem -O ue_urllc.conf --sa -r 106 --numerology 1 -E --continuous-tx --uicc0.imsi 001010000000002

# 5. Confirm both UEs attached and got a tunnel IP
ip a | grep oaitun     # run on each UE PC — should show 10.0.2.17 (UE1) / 10.0.0.11 (UE2)

# 6. On the Core PC — start iperf3 servers, one per slice
docker exec -d oai-ext-dn iperf3 -s -p 5201
docker exec -d oai-ext-dn iperf3 -s -p 5202

# 7. On each UE PC — run the iperf3 client (see "Testing throughput per slice" for full commands)
```

If anything above fails, jump to [Step-by-Step Setup](#step-by-step-setup) or the [Troubleshooting log](#troubleshooting-log).

## Architecture

| Role | Machine | Notes |
|---|---|---|
| OAI 5G Core + gNB | PC1 | Runs all core NF containers + gNB. Hosts `oai-ext-dn` (Data Network) at `192.168.70.135`. |
| UE1 (eMBB) | PC2 | Own USRP. Tunnel IP `10.0.2.17` after PDU session establishment. |
| UE2 (URLLC) | PC3 | Own USRP. Tunnel IP `10.0.0.11` after PDU session establishment. |

All three machines communicate over their USRPs (RF sync confirmed) before any slice-level testing.

### Core container network (`docker-compose.yaml`)

| Container | IP | Purpose |
|---|---|---|
| mysql | 192.168.70.131 | Subscriber DB |
| oai-nrf | 192.168.70.130 | NF Repository Function |
| oai-amf | 192.168.70.132 | Access & Mobility Function (N2: 38412/sctp) |
| oai-smf | 192.168.70.133 | Session Management Function |
| oai-upf | 192.168.70.134 | User Plane Function |
| oai-udr | 192.168.70.136 | Unified Data Repository |
| oai-udm | 192.168.70.137 | Unified Data Management |
| oai-ausf | 192.168.70.138 | Authentication Server Function |
| ims | 192.168.70.139 | Asterisk-based IMS (SIP) |
| oai-ext-dn | 192.168.70.135 | External Data Network — iperf3 server endpoint used in testing |

## Repo structure

```
.
├── README.md
├── docker-compose.yaml        # Full core stack: NFs, mysql, ims, ext-dn
├── conf/
│   ├── config.yaml            # Shared OAI CN config (AMF/SMF/UPF/slices/DNNs)
│   ├── sip.conf                # Asterisk IMS SIP config
│   └── users.conf              # IMS SIP users (one per UE IMSI)
├── ue-conf/
│   ├── ue_embb.conf            # UE1 — eMBB slice
│   └── ue_urllc.conf           # UE2 — URLLC slice
└── docs/
    └── OAI_MultiUE_Slicing_Testbed_Documentation.docx
```

## Network slice configuration

Two UEs are provisioned on distinct slices, differentiated by IMSI, DNN, and NSSAI (SST = Slice/Service Type, SD = Slice Differentiator).

| UE | IMSI | DNN | SST | SD |
|---|---|---|---|---|
| UE1 (eMBB) | 001010000000001 | `oai` | 1 | 0xFFFFFF |
| UE2 (URLLC) | 001010000000002 | `lowlat` | 2 | 0x010203 |

`ue_embb.conf`:
```ini
uicc0 = {
  imsi = "001010000000001";
  key = "fec86ba6eb707ed08905757b1bb44b8f";
  opc = "C42449363BBAD02B66D16BC975D77CC1";
  dnn = "oai";
  nssai_sst = 1;
  nssai_sd = 0xFFFFFF;
  pdu_sessions = (
    {
       pdu_session_id = 1;
       pdu_type = "IPV4";
       dnn = "oai";
       nssai_sst = 1;
       nssai_sd = 0xFFFFFF;
    }
  );
};
```

`ue_urllc.conf`:
```ini
uicc0 = {
  imsi = "001010000000002";
  key = "fec86ba6eb707ed08905757b1bb44b8f";
  opc = "C42449363BBAD02B66D16BC975D77CC1";
  dnn = "lowlat";
  nssai_sst = 2;
  nssai_sd = 0x010203;
  pdu_sessions = (
    {
       pdu_session_id = 1;
       pdu_type = "IPV4";
       dnn = "lowlat";
       nssai_sst = 2;
       nssai_sd = 0x010203;
    }
  );
};
```

## Step-by-Step Setup

This section walks through the full setup from a clean PC to a working multi-slice attach, across all three machines. Skip to whichever step matches where you're starting from.

### 0. Prerequisites (all 3 PCs)

| Requirement | Notes |
|---|---|
| OS | Ubuntu 22.04 LTS (recommended by OAI for this RAN/CN release) |
| USRP | One per PC (e.g. B210/B205mini/X3xx) with UHD drivers installed and `uhd_find_devices` showing the device |
| Docker + Docker Compose | Core PC only — used to run all core NFs |
| OAI RAN repo (`openairinterface5g`) | Core PC (for gNB) + both UE PCs (for nrUE) — build with `--gNB` / `--nrUE` flags respectively |
| OAI CN5G repo (this repo's `docker-compose.yaml` is based on it) | Core PC only |
| Network connectivity between all 3 PCs | Needed so the UEs can reach the gNB's IP over their RF link path / fronthaul config, and so the Core PC's Docker network is reachable if any host-network bridging is used |

If you haven't built OAI yet, follow the official build instructions before continuing:
- RAN (gNB/UE): https://gitlab.eurecom.fr/oai/openairinterface5g
- CN5G (core): https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed

### 1. Clone this repo (Core PC)

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

### 2. Provision subscribers in the core DB

The two IMSIs in [Network slice configuration](#network-slice-configuration) must exist in the core's subscriber database (mysql) before either UE can attach. If you're using the OAI CN5G default `oai_db.sql`, edit it (or insert via the UDR/UDM API) to add both IMSIs with their slice (NSSAI) and DNN assignments matching the table above, **before** first bringing up the stack.

### 3. Bring up the core (Core PC)

```bash
docker compose up -d
docker compose ps        # confirm all NFs + mysql + ims + ext-dn are healthy
```

Wait until every container shows `healthy` (not just `running`) before proceeding — `oai-amf` and `oai-smf` in particular need `oai-nrf` and `mysql` to be ready first.

### 4. Start the gNB (Core PC)

With the core containers up, start the gNB process (on the host, not in a container) using your gNB config (the `conf/config.yaml` in this repo describes the matching CN-side slice/DNN setup — your gNB conf should reference the same SST/SD values):

```bash
sudo ./nr-softmodem   -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf   --gNBs.[0].min_rxtxtime 8 -E
```

Confirm in the gNB logs that it registers with the AMF (`N2 setup` success) before starting either UE.

### 5. Start UE1 — eMBB (UE1's PC)

```bash
sudo "./nr-uesoftmodem" "-r" "106" "--numerology" "1" "--band" "78" "-C" "3619200000" "--ssb" "516" "-E" "-O" "/<Path-of-the-ue_embb.conf>/ue_embb.conf"
```

Watch the logs for `RRC_CONNECTED` and a successful PDU session establishment. On success, a `oaitun_ue1`-style interface appears:

```bash
ip a | grep oaitun     # should show 10.0.2.17
```

### 6. Start UE2 — URLLC (UE2's PC)

```bash
 sudo "./nr-uesoftmodem" "-r" "106" "--numerology" "1" "--band" "78" "-C" "3619200000" "--ssb" "516" "-E" "-O" "/<Path-of-the-ue_urllc.conf>/ue_urllc.conf"


```

Same check:

```bash
ip a | grep oaitun     # should show 10.0.0.11
```

### 7. Verify both slices are up

On the Core PC, check that the SMF/UPF logs show two separate PDU sessions, one per DNN (`oai` and `lowlat`). Then proceed to [Testing throughput per slice](#testing-throughput-per-slice).

> **Tip:** bring up one UE at a time the first time through. If UE1 attaches but UE2 doesn't, the issue is almost always in `ue_urllc.conf`'s NSSAI/DNN values not matching what's provisioned in the core DB — re-check step 2.

## Testing throughput per slice

iperf3 servers run inside `oai-ext-dn`; each UE PC runs the client bound to its own tunnel IP.

```bash
# On the core PC — start one server per slice/port
docker exec -d oai-ext-dn iperf3 -s -p 5201
docker exec -d oai-ext-dn iperf3 -s -p 5202

# On UE1's PC (eMBB)
iperf3 -c 192.168.70.135 -B 10.0.2.17 -u -b 2M -R -t 30 -p 5202

# On UE2's PC (URLLC)
iperf3 -c 192.168.70.135 -B 10.0.0.11 -u -b 2M -R -t 30 -p 5201
```

### Results

| | UE1 (eMBB) | UE2 (URLLC) |
|---|---|---|
| Bound IP | 10.0.2.17 | 10.0.0.11 |
| Bitrate | 2.00 Mbits/sec | 2.00 Mbits/sec |
| Jitter | 2.552 ms | 7.832 ms |
| Loss | 0/5189 (0%) | 0/5180 (0%) |

## Troubleshooting log

| Issue | Cause | Fix |
|---|---|---|
| `iperf3: error - unable to start listener... Address already in use` | Stale iperf3 process still bound to the port | `pkill -9 iperf3` before relaunching the server |
| `bash: dev/null: No such file or directory` | Missing leading slash (`2>dev/null` instead of `2>/dev/null`) | Use `2>/dev/null` |
| `unknown shorthand flag: 's' in -s` | Typo: `docker exec-d` instead of `docker exec -d` (missing space) | `docker exec -d oai-ext-dn iperf3 -s -p 5201` |
| `Cannot assign requested address` on iperf3 client | Client run from the core PC, bound (`-B`) to an IP it doesn't own (the UE's tunnel IP) | Run each client **on its own UE PC**, bound to its own real interface IP |

> `-B` always binds to a local address on the machine the command runs on — when testing multiple UEs on separate PCs, run the iperf3 client on each UE's own PC, not centrally from the core.

## Known notes / lab-only credentials

This repo contains lab-only placeholder credentials (MySQL `test`/`test`, root `linux`). Replace these before using outside an isolated test network.

## License

OAI components are licensed under the OAI Public License v1.1 — see [openairinterface.org](http://www.openairinterface.org/?page_id=698).
