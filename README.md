# OAI 5G Multi-UE Network Slicing Testbed (eMBB + URLLC)

A 3-PC, 3-USRP OpenAirInterface (OAI) 5G testbed running a single Core + gNB instance that serves **two UEs on two separate network slices** — eMBB and URLLC — verified end-to-end with iperf3.

---

## Contents

- [Description](#descroption)
- [Architecture](#architecture)
- [Repo Structure](#repo-structure)
- [Network Slice Configuration](#network-slice-configuration)
- [Step-by-Step Setup](#step-by-step-setup)
- [Testing Throughput per Slice](#testing-throughput-per-slice)
- [Troubleshooting Log](#troubleshooting-log)
- [Known Notes / Lab-Only Credentials](#known-notes--lab-only-credentials)
- [License](#license)

---

## Description

✅ Multi-UE, multi-slice operation confirmed working. Both UEs attached, established PDU sessions on their respective slices, and exchanged UDP traffic with the core's external Data Network container at 2 Mbit/s with **0% packet loss** over a 30-second test.

---


## Architecture

| Role              | Machine | Notes                                                                                      |
|-------------------|---------|--------------------------------------------------------------------------------------------|
| OAI 5G Core + gNB | PC1     | Runs all core NF containers + gNB. Hosts `oai-ext-dn` (Data Network) at `192.168.70.135`. |
| UE1 (eMBB)        | PC2     | Own USRP B210. Tunnel IP `10.0.2.17` after PDU session establishment.                     |
| UE2 (URLLC)       | PC3     | Own USRP B210. Tunnel IP `10.0.0.11` after PDU session establishment.                     |

All three machines communicate over their USRPs (RF sync confirmed) before any slice-level testing.

### Core Container Network

| Container  | IP             | Purpose                                                        |
|------------|----------------|----------------------------------------------------------------|
| mysql      | 192.168.70.131 | Subscriber DB                                                  |
| oai-nrf    | 192.168.70.130 | NF Repository Function                                         |
| oai-amf    | 192.168.70.132 | Access & Mobility Function (N2: 38412/sctp)                    |
| oai-smf    | 192.168.70.133 | Session Management Function                                    |
| oai-upf    | 192.168.70.134 | User Plane Function                                            |
| oai-udr    | 192.168.70.136 | Unified Data Repository                                        |
| oai-udm    | 192.168.70.137 | Unified Data Management                                        |
| oai-ausf   | 192.168.70.138 | Authentication Server Function                                 |
| ims        | 192.168.70.139 | Asterisk-based IMS (SIP)                                       |
| oai-ext-dn | 192.168.70.135 | External Data Network — iperf3 server endpoint used in testing |

---

## Repo Structure

This is the exact file structure of this repository:

```
Dual_UE_Operation_by_using_OAI/
├── README.md
└── repo/
    └── configs/
        ├── ue.conf  
          └── ue_embb.conf      # UE1 config — eMBB slice (SST=1, DNN=oai)
          └── ue_urllc.conf     # UE2 config — URLLC slice (SST=2, DNN=lowlat)
        ├── oai-cn5g
         └── conf
           └── config.yml
           └── sip.conf
           └──users.conf
        ├── docker-compose.yml
        ├── mysql-healthcheck.sh
        ├── oai_db.sql  
```

> The `docker-compose.yaml` for the 5G core and the gNB `.conf` file are sourced from the official [oai-cn5g-fed](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed) and [openairinterface5g](https://gitlab.eurecom.fr/oai/openairinterface5g) repositories respectively. This repo stores only the UE-side slice configs under `repo/configs/`.

---

## Network Slice Configuration

Two UEs are provisioned on distinct slices, differentiated by IMSI, DNN, and NSSAI (SST = Slice/Service Type, SD = Slice Differentiator).

| UE          | IMSI            | DNN      | SST | SD       |
|-------------|-----------------|----------|-----|----------|
| UE1 (eMBB)  | 001010000000001 | `oai`    | 1   | 0xFFFFFF |
| UE2 (URLLC) | 001010000000002 | `lowlat` | 2   | 0x010203 |

**`repo/configs/ue_embb.conf`:**
```
uicc0 = {
  imsi = "001010000000001";
  key  = "fec86ba6eb707ed08905757b1bb44b8f";
  opc  = "C42449363BBAD02B66D16BC975D77CC1";
  dnn  = "oai";
  nssai_sst = 1;
  nssai_sd  = 0xFFFFFF;
  pdu_sessions = (
    {
      pdu_session_id = 1;
      pdu_type       = "IPV4";
       dnn            = "oai";
      nssai_sst      = 1;
      nssai_sd       = 0xFFFFFF;
    }
  );
};
```

**`repo/configs/ue_urllc.conf`:**
```
uicc0 = {
  imsi = "001010000000002";
  key  = "fec86ba6eb707ed08905757b1bb44b8f";
  opc  = "C42449363BBAD02B66D16BC975D77CC1";
  dnn  = "lowlat";
  nssai_sst = 2;
  nssai_sd  = 0x010203;
  pdu_sessions = (
    {
      pdu_session_id = 1;
      pdu_type       = "IPV4";
       dnn            = "lowlat";
      nssai_sst      = 2;
      nssai_sd       = 0x010203;
    }
  );
};
```

---

## Step-by-Step Setup

This section walks through the full setup from a clean PC to a working multi-slice attach, across all three machines. Skip to whichever step matches where you are starting from.

### 0. Prerequisites (all 3 PCs)

| Requirement                           | Notes                                                                                                                              |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| OS                                    | Ubuntu 22.04 LTS (recommended by OAI for this RAN/CN release)                                                                     |
| USRP                                  | One per PC (B210/B205mini/X3xx) with UHD drivers installed — verify with `uhd_find_devices`                                       |
| Docker + Docker Compose               | Core PC (PC1) only — used to run all core NFs                                                                                      |
| OAI RAN repo (`openairinterface5g`)   | PC1 (for gNB) + PC2/PC3 (for nrUE) — build with `--gNB` / `--nrUE` flags respectively                                            |
| OAI CN5G (`oai-cn5g-fed`)            | PC1 only                                                                                                                           |
| Network connectivity between all 3 PCs | Required so UE PCs can reach PC1's Docker bridge subnet (`192.168.70.128/26`) for data plane traffic through the UPF              |

Official build references:
- RAN (gNB/UE): https://gitlab.eurecom.fr/oai/openairinterface5g
- CN5G (core): https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-fed

---

### 1. Clone this repo (PC1 — Core)

```bash
git clone https://github.com/Niladri-Roy07/Dual_UE_Operation_by_using_OAI.git
cd Dual_UE_Operation_by_using_OAI
```

---
put the congiuration files in the corresponding folders and restart the docker containers using below comments.

### 2. Provision subscribers in the core DB (PC1)

The two IMSIs in the [Network Slice Configuration](#network-slice-configuration) table must exist in the MySQL subscriber database before either UE can attach. Using the OAI CN5G default `oai_db.sql`, insert both IMSIs with their matching DNN and NSSAI values **before** first bringing up the stack.

---

### 3. Bring up the core (PC1)

```bash
docker compose down    # stop the core network
docker compose up -d
docker compose ps      # confirm all NFs + mysql + ims + ext-dn are healthy

```
![Core containers healthy](repo/output_images/01_core_docker_compose_up.png)

> ⚠️ Wait until every container shows `healthy` — not just `running`. `oai-amf` and `oai-smf` depend on `oai-nrf` and `mysql` being fully ready first.

---
put the gnb configuration file (gnb.sa.band78.fr1.106PRB.usrpb210.conf) in the corresponding folder (/openairinterface5g/targets/PROJECTS/GENERIC-NR-5GC/CONF) and then run the following command
### 4. Start the gNB (PC1)

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem   -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf   --gNBs.[0].min_rxtxtime 8 -E

```

Confirm `N2 setup` success in the gNB logs before starting either UE.

---
put the ue configuration file (ue_embb.conf) in the corresponding folder (/openairinterface5g/cmake_targets/ran_build/build) in the PC2 and then run the following command
### 5. Start UE1 — eMBB (PC2)

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo "./nr-uesoftmodem" "-r" "106" "--numerology" "1" "--band" "78" "-C" "3619200000" "--ssb" "516" "-E" "-O" "ue_embb.conf"
```

Watch the logs for `RRC_CONNECTED` and a successful PDU session. On success the tunnel interface appears:

```bash
ip a | grep oaitun     # should show 10.0.2.17
```

---
put the ue configuration file (ue_urllc.conf) in the corresponding folder (/openairinterface5g/cmake_targets/ran_build/build) in the PC3 and then run the following command
### 6. Start UE2 — URLLC (PC3)

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo "./nr-uesoftmodem" "-r" "106" "--numerology" "1" "--band" "78" "-C" "3619200000" "--ssb" "516" "-E" "-O" "ue_urllc.conf"
```

```bash
ip a | grep oaitun     # should show 10.0.0.11
```
```bash
ping 192.168.70.129    # IN UE1 and UE2 as well
```

> **Tip:** Bring up one UE at a time on the first run. If UE1 attaches but UE2 doesn't, the cause is almost always `ue_urllc.conf`'s NSSAI/DNN values not matching what was provisioned in the core DB — re-check step 2.

---

### 7. Verify both slices are up (PC1)

Check that SMF/UPF logs show two separate PDU sessions — one for DNN `oai` and one for `lowlat`. Then proceed to traffic testing.

---

## Testing Throughput per Slice

iperf3 servers run inside `oai-ext-dn` on PC1. Each UE PC runs its own iperf3 client bound to its tunnel IP.

**PC1 — start one server per slice/port:**
```bash
# Always kill stale iperf3 processes first
docker exec oai-ext-dn pkill -9 iperf3 2>/dev/null
sleep 1
docker exec -d oai-ext-dn iperf3 -s -p 5201
docker exec -d oai-ext-dn iperf3 -s -p 5202
```

**PC2 — UE1 (eMBB) client:**
```bash
iperf3 -c 192.168.70.135 -B 10.0.2.17 -u -b 2M -R -t 30 -p 5202
```

**PC3 — UE2 (URLLC) client:**
```bash
iperf3 -c 192.168.70.135 -B 10.0.0.11 -u -b 2M -R -t 30 -p 5201
```

> **Important:** The `-B` flag binds the socket to a local IP address on the machine running the command. Always run each UE's iperf3 client **on its own PC** — the PC that actually owns that tunnel IP.

### Results

|          | UE1 (eMBB)     | UE2 (URLLC)    |
|----------|----------------|----------------|
| Bound IP | 10.0.2.17      | 10.0.0.11      |
| Bitrate  | 2.00 Mbits/sec | 2.00 Mbits/sec |
| Jitter   | 2.552 ms       | 7.832 ms       |
| Loss     | 0/5189 (0%)    | 0/5180 (0%)    |

---

## Troubleshooting Log

These are real issues hit during bring-up on **PC2 (UE1 / eMBB machine)**, with the exact terminal output and the fix applied each time.

---

### Issue 1 — iperf3: Address Already in Use

**Exact error:**
```
iperf3: error - unable to start listener for connections: Address already in use
iperf3: exiting
```

**What happened:**
A stale iperf3 server process from a previous run was still bound to port 5201 or 5202 inside the `oai-ext-dn` container. Launching new servers without cleaning up the old ones caused immediate failure. This happened twice in sequence — the first attempt failed on both ports, the second attempt succeeded on 5201 but still failed on 5202 because something was still holding it.

**Fix:**
```bash
# Kill all stale iperf3 processes inside the container first
docker exec oai-ext-dn pkill -9 iperf3 2>/dev/null
sleep 1
docker exec -d oai-ext-dn iperf3 -s -p 5201
docker exec -d oai-ext-dn iperf3 -s -p 5202
```

Always run the `pkill` step before starting iperf3 servers — especially after any previous test or failed attempt.

---

### Issue 2 — bash: dev/null: No such file or directory

**Exact error:**
```
docker exec oai-ext-dn pkill -9 iperf3 2>dev/null
bash: dev/null: No such file or directory
```

**What happened:**
A missing leading `/` in the stderr redirect. `2>dev/null` tells the shell to redirect stderr into a file literally named `dev/null` in the current working directory (`~/openairinterface5g/cmake_targets/ran_build/build/`). That file does not exist, so the shell threw an error rather than discarding the output silently.

**Fix:**
```bash
# Correct — redirects stderr to the null device (discards it)
docker exec oai-ext-dn pkill -9 iperf3 2>/dev/null
```

---


### Issue 3 — iperf3 Client: Cannot Assign Requested Address

**Exact error:**
```
iperf3: error - unable to connect to server - server may have stopped running
or use a different port, firewall issue, etc.: Cannot assign requested address
```

**What happened:**
The iperf3 clients were launched from the core PC (PC1) with `-B 10.0.0.2`, `-B 10.0.0.3`, `-B 10.0.0.4` — IP addresses that do not belong to any interface on PC1. The `-B` flag tells iperf3 to bind its local socket to a specific IP. If that IP is not assigned to any interface on the machine running the command, the OS refuses the bind and throws this error.

Additionally, the very first iperf3 command had a bracketed-paste terminal artifact (`^[[200~`) prepended to it from pasting into the terminal, which caused:
```
iperf3: command not found
```

**Fix:**
Run each iperf3 client **on the PC that actually owns that tunnel IP**:

```bash
# On PC2 — this PC owns 10.0.2.17, so run the client here
iperf3 -c 192.168.70.135 -B 10.0.2.17 -u -b 2M -R -t 30 -p 5202

# On PC3 — this PC owns 10.0.0.11, so run the client here
iperf3 -c 192.168.70.135 -B 10.0.0.11 -u -b 2M -R -t 30 -p 5201
```

For the bracketed-paste issue: type commands manually or paste into a plain text editor first to strip escape sequences before running in the terminal.

---

### Issue 4 — Ping: Destination Host Unreachable (Wrong Default Route on PC1)

**Exact error on PC1:**
```
ping 10.0.0.8
# Result:
PING 10.0.0.8 (10.0.0.8) 56(84) bytes of data.
From 192.168.70.129 icmp_seq=1 Destination Host Unreachable
From 192.168.70.129 icmp_seq=2 Destination Host Unreachable
...
16 packets transmitted, 0 received, +13 errors, 100% packet loss
```

**What happened:**
PC2's default route was pointing to `192.168.70.134` — the UPF container's IP — via the `oai-cn5g` Docker bridge interface. This was the wrong gateway. The routing table at the time of failure:

```
ip route show
# Result:
default via 192.168.70.134 dev oai-cn5g    ← wrong: UPF IP is not a host gateway
default via 172.16.128.1 dev wlp2s0 ...
192.168.70.128/26 dev oai-cn5g proto kernel scope link src 192.168.70.129
```

The UPF (`192.168.70.134`) is a container-internal endpoint, not a gateway router for the Docker bridge subnet. Traffic destined for `10.0.0.x` routed through it had no valid return path from the host side. Every ICMP packet came back as `Destination Host Unreachable` from the Docker bridge gateway (`192.168.70.129`).

**Fix — corrected the default route on PC1(core+gnb):**
```bash
# Step 1: Remove the wrong default route pointing to the UPF
sudo ip route del default via 192.168.70.134 dev oai-cn5g

# Step 2: Add the correct default via the Docker bridge's own gateway
sudo ip route add default via 192.168.70.129 dev oai-cn5g
```

**Resulting correct routing table on PC1(core+gnb):**
```
ip route show
# Results:
default via 192.168.70.129 dev oai-cn5g          ← correct gateway
default via 172.16.128.1 dev wlp2s0 proto dhcp src 172.16.139.175 metric 600
172.16.128.0/20 dev wlp2s0 proto kernel scope link src 172.16.139.175 metric 600
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.70.128/26 dev oai-cn5g proto kernel scope link src 192.168.70.129
```
IN PC3 (UE2):
```
sudo ip route add default via 10.0.0.0 dev oaitun_ue1
```
IN PC2 (UE1):
```
sudo ip route add default via 10.0.2.0 dev oaitun_ue1
```
IN PC2(UE1) and PC3(UE2) both:
```
ip route show
# Results:
default via 10.0.2.0 dev oaitun_ue1                                           #IN PC PC3
default via 172.16.176.1 dev wlp2s0 proto dhcp src 172.16.177.7 metric 600 
10.0.2.0/24 dev oaitun_ue1 proto kernel scope link src 10.0.2.2 
172.16.176.0/22 dev wlp2s0 proto kernel scope link src 172.16.177.7 metric 600 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown    

default via 10.0.0.0 dev oaitun_ue1                                          #IN PC PC2
default via 172.16.124.1 dev wlp14s0 proto dhcp metric 600 
10.0.0.0/24 dev oaitun_ue1 proto kernel scope link src 10.0.0.2 
169.254.0.0/16 dev wlp14s0 scope link metric 1000 
172.16.124.0/23 dev wlp14s0 proto kernel scope link src 172.16.124.49 metric 600 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
```
> **Note:** Even after fixing the routing, pings to `10.0.0.x` from the host still return unreachable — and that is **expected**. Those IPs live on the UE's `oaitun_ue1` tunnel interface, not on the host machine. The correct end-to-end reachability test is the iperf3 test bound to the tunnel IP and run from within the UE's own PC, which is exactly what the [Testing Throughput](#testing-throughput-per-slice) section does.

---

## Known Notes / Lab-Only Credentials

This repo was built and tested in an isolated lab environment. It contains placeholder credentials:

| Item           | Value   |
|----------------|---------|
| MySQL user     | `test`  |
| MySQL password | `test`  |
| Root password  | `linux` |

**Replace all credentials before deploying outside an air-gapped test network.**

---

## License

OAI components are licensed under the OAI Public License v1.1 — see [openairinterface.org](http://www.openairinterface.org/?page_id=698).
