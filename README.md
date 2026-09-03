# Slot 15 — O-RAN CU–DU Split Testbed: F1 Interface Message Analysis

A CU/DU-split gNB testbed built with srsRAN Project (commit `ef4b0749a1`) running against a ZMQ virtual RF, backed by a full Open5GS core, with UERANSIM/srsUE used as the UE side. The goal was to capture and analyse F1AP signalling — both the F1 Setup procedure between CU and DU, and (as a stretch goal) UE-context signalling once a UE attaches.

## Repository structure

```
github_project/
├── README.md
├── .gitignore
├── configs/
│   ├── cu.yml                     # CU-CP config (F1-C + N2/AMF)
│   ├── du_zmq.yml                 # DU config, ZMQ virtual RF
│   ├── amf.yaml                   # Open5GS AMF config
│   └── open5gs-ue.example.yaml    # UE subscriber profile, credentials redacted
├── captures/
│   └── f1ap_live.pcap             # SCTP + F1AP capture (F1SetupRequest/Response)
└── docs/
    ├── f1c_sctp_connection.txt    # SCTP association evidence
    ├── f1ap_packet_summary.txt    # tshark/wireshark packet summary
    ├── network_ports.txt          # ss/netstat listener evidence
    ├── sctp_connections.txt       # SCTP endpoint state
    └── software_versions.txt      # srsRAN commit, Open5GS version, UERANSIM version
```

## What's running

| Component | Role | Address |
|---|---|---|
| Open5GS AMF | 5G core, N2 termination | `127.0.1.100:38412` |
| srsRAN CU (`srscu`) | CU-CP: F1-C termination, N2 to AMF | F1-C on `127.0.10.1` |
| srsRAN DU (`srsdu`) | DU: F1-C client, ZMQ-backed cell | F1-C client at `127.0.10.2`, PHY over ZMQ |
| UERANSIM / srsUE | UE simulator over ZMQ | ZMQ TCP ports `2000`/`2001` |
| Wireshark / tcpdump | F1AP + SCTP capture | loopback, F1-C port |

## Setup

### 1. Core network
Open5GS is deployed as systemd services with the AMF config at `configs/amf.yaml`. The AMF's N2 SCTP socket comes up on `127.0.1.100:38412`, verified with `ss -lnp | grep sctp` before anything RAN-side is started.

### 2. CU
`configs/cu.yml` is the actual config used:

```yaml
cu_cp:
  amf:
    addr: 127.0.1.100
    port: 38412
    bind_addr: 127.0.10.2
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "99970"
            tai_slice_support_list:
              - sst: 1
  f1ap:
    bind_addr: 127.0.10.1

cu_up:
  nru:
    bind_addr: 127.0.10.1

log:
  filename: /tmp/cu.log
  all_level: warning
```

Started with:

```bash
./build/apps/cu/srscu -c configs/cu.yml
```

F1-C comes up listening on `127.0.10.1`, and the CU registers to the AMF over N2.
srsRAN Project — CU

Built from the release_24_10_1 tag (commit ef4b0749a1). The processes, run in separate terminals:

bash
./build/apps/cu/srscu -c configs/cu.yml

CU-CP listens for F1-C on 127.0.10.1:38472 and connects north to the AMF on 127.0.1.100:38412. 

### 3. DU
The shipped example config (`du_rf_b200_tdd_n78_20mhz.yml`) targets a physical USRP B200 over UHD — not usable for a virtual-RF testbed. It was copied to `configs/du_zmq.yml` and the `ru_sdr` block was rebuilt for ZMQ instead of guessing the argument syntax from a different srsRAN version: the actual argument names were confirmed by inspecting `lib/radio/zmq/radio_session_zmq_impl.cpp` and `radio_zmq_baseband_gateway.h` directly in the `ef4b0749a1` source tree, since `--device_args help` on this build doesn't print usage — it proceeds straight into initialization and fails (`ZMQ transmission channel arguments out of bounds`) if the args aren't already correct.

```bash
./build/apps/du/srsdu -c configs/du_zmq.yml
```

DU's F1-C connects to the CU-CP on the same F1-C address and comes up with a ZMQ-backed n78 cell (bw=20 MHz, dl_arfcn=650000, dl_freq=3750.0 MHz, pci=1, 1T1R), and the F1 Setup procedure completes.

### 4. UE
UERANSIM's `nr-ue` and srsRAN_4G's `srsue` were both tried against the DU's ZMQ endpoint (TCP ports `2000`/`2001`). Confirming the ZMQ data path is actually established (not just listening) is done with:

```bash
ss -tnp | grep -E '2000|2001'
```
### 5. Capturing F1
bash
sudo tcpdump -i lo -nn -s 0 -w ~/f1ap_live.pcap 'sctp port 38472'

## Results
Full SCTP four-way association (INIT / INIT ACK / COOKIE ECHO / COOKIE ACK) between DU (127.0.10.2:37628) and CU-CP (127.0.10.1:38472).
- **F1-C SCTP association**: established between DU and CU-CP, evidence in `docs/f1c_sctp_connection.txt` and `docs/sctp_connections.txt`.
- **F1SetupRequest / F1SetupResponse**: captured in `captures/f1ap_live.pcap`, summarised in `docs/f1ap_packet_summary.txt`.

Periodic SCTP HEARTBEAT/HEARTBEAT ACK keeping the association alive.


## References

- 3GPP TS 38.401 / TS 38.470 / TS 38.473 — NG-RAN architecture, F1 general aspects, F1AP
- O-RAN Alliance Architecture Description — RU/DU/CU functional splits
- srsRAN Project documentation (https://docs.srsran.com)
- UERANSIM documentation (https://github.com/aligungr/UERANSIM)
- Open5GS documentation (https://open5gs.org/open5gs/docs/)
- RFC 4960 — Stream Control Transmission Protocol
