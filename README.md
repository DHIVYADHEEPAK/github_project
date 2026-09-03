# Slot 15 — O-RAN CU–DU Split Testbed: F1 Interface Message Analysis

A hands-on 5G RAN testbed built to disaggregate the gNB into CU and DU components per the O-RAN split architecture, and to capture and read the F1AP setup signalling that the two components exchange over F1-C. The DU is driven over a ZMQ virtual radio link, the core is a full Open5GS deployment, and the UE side is exercised with UERANSIM (with a parallel srsUE/4G run kept for comparison).

This repo documents the deployment, the config issues that came up, and a byte-level read of the captured F1AP `F1SetupRequest` / `F1SetupResponse` exchange.

## What's actually in this testbed

| Component | Role | Binary / Service |
|---|---|---|
| Open5GS | 5GC (AMF, SMF, UPF, AUSF, UDM, NRF, SCP, SEPP, BSF, plus legacy SGW-C/U, MME for the LTE comparison run) | `open5gs-*d` systemd services |
| srsRAN Project (`release_24_10_1`, commit `ef4b0749a1`) | gNB split into CU-CP/CU-UP and DU | `apps/cu/srscu`, `apps/du/srsdu` |
| ZMQ virtual RF | Loopback RF between DU and UE, no SDR needed | `libsrsran_rf_zmq.so` |
| UERANSIM | NR UE simulator | `nr-ue`, configs for open5gs / free5gc / custom |
| srsRAN_4G (`srsue`) | LTE UE, used in a side-by-side attach test over ZMQ | `srsue/src/srsue` |
| Wireshark / tcpdump | F1AP + SCTP capture and dissection | `f1ap_live.pcap`, `f1u_live.pcap` |

Everything runs on loopback (`127.0.10.x` for F1, `127.0.1.100` / `127.0.0.x` for N2 and the 5GC internal SBI/Diameter interfaces) on a single Ubuntu host — no physical RAN hardware involved, which is the point of the ZMQ split.

## Repo layout

```
.
├── README.md
├── docs/
│   ├── PROJECT_REPORT.md      # full write-up: deployment notes + F1AP analysis
│   └── REFERENCES.md
├── configs/
│   ├── cu.yml                 # CU-CP/CU-UP config used against Open5GS
│   ├── du_zmq.yml             # DU config, ZMQ RF, n78 cell
│   └── ue_zmq.conf.snippet    # srsue ZMQ carrier settings (LTE comparison run)
└── captures/
    └── README.md              # what f1ap_live.pcap / f1u_live.pcap contain, filters used
```

## Setup

### 1. Core network — Open5GS
Open5GS was installed as systemd services and left running as the full NF set (AMF, AUSF, BSF, MME, NRF, NSSF, SCP, SEPP, SGW-C, SGW-U, SMF, UDM, UPF):

```bash
sudo systemctl --type=service --state=running | grep open5gs
```

AMF's N2 SCTP socket comes up on `127.0.1.100:38412`, confirmed with:

```bash
sudo ss -lnp | grep -i sctp
```

### 2. srsRAN Project — CU and DU
Built from the `release_24_10_1` tag (commit `ef4b0749a1`). Two processes, run in separate terminals:

```bash
./build/apps/cu/srscu -c configs/cu.yml
./build/apps/du/srsdu -c configs/du_zmq.yml
```

CU-CP listens for F1-C on `127.0.10.1:38472` and connects north to the AMF on `127.0.1.100:38412`. DU connects to the CU-CP on the same F1-C address and comes up with a ZMQ-backed n78 cell (`bw=20 MHz`, `dl_arfcn=650000`, `dl_freq=3750.0 MHz`, `pci=1`, `1T1R`).

### 3. UE
UERANSIM's `nr-ue` was pointed at the `open5gs-ue.yaml` config (`~/UERANSIM/config/open5gs-ue.yaml`) to attach through the DU/CU pair. Separately, srsRAN_4G's `srsue` was run against a ZMQ config with the NR carrier disabled (`rat.eutra.nof_carriers=0`, `dl_earfcn=3350`) purely to sanity-check the ZMQ loopback plumbing against an LTE stack before trusting the 5G NR results.

### 4. Capturing F1
```bash
sudo tcpdump -i lo -nn -s 0 -w ~/f1ap_live.pcap 'sctp port 38472'
```
(Writing to `/tmp` failed with `Permission denied` under the sudo tcpdump context on this box — redirected the capture to the home directory instead.)

## Results

- Full SCTP four-way association (INIT / INIT ACK / COOKIE ECHO / COOKIE ACK) between DU (`127.0.10.2:37628`) and CU-CP (`127.0.10.1:38472`).
- `F1SetupRequest` (246 bytes) and `F1SetupResponse` (118 bytes) captured and decoded in Wireshark, including the `gNB-DU-Name` (`srsdu0`) and `gNB-CU-Name` (`cu_cp_01`) IEs.
- Periodic SCTP HEARTBEAT/HEARTBEAT ACK keeping the association alive.
- F1-U (GTP-U) capture came back empty for the `gtp` filter — see `docs/PROJECT_REPORT.md` for why that's a meaningful negative result, not a capture bug.

Full byte-level walkthrough, the functional-split explanation, and the config-parsing error hit along the way are in [`docs/PROJECT_REPORT.md`](docs/PROJECT_REPORT.md).

## References

See [`docs/REFERENCES.md`](docs/REFERENCES.md).
