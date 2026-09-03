# ORAN CU-DU Split Testbed – F1 Interface Message Analysis
 
## Project Overview
 
This project demonstrates an O-RAN gNB CU-DU split using srsRAN.
 
The CU and DU communicate over the F1 interface. The F1-C control plane

uses SCTP transport and F1AP signalling.
 
## Architecture
 
UE / RAN simulation

        |

        v

      DU

        |

        | F1-C / SCTP

        | 127.0.10.2:47144

        |

        v

      CU

        |

        | N2 / NGAP

        v

     Open5GS AMF
 
## Software Used
 
- srsRAN

- Open5GS

- UERANSIM

- Wireshark

- tcpdump

- Linux / Ubuntu
 
## F1-C Configuration
 
CU F1-C address:
 
127.0.10.1:38472
 
DU F1-C address:
 
127.0.10.2
 
Transport:
 
SCTP
 
## Verified Results
 
The CU-DU F1-C SCTP association was successfully established.
 
Evidence is available in:
 
docs/f1c_sctp_connection.txt
 
Packet capture:
 
captures/f1ap_live.pcap
 
Packet summary:
 
docs/f1ap_packet_summary.txt
 
Wireshark analysis identified F1AP setup signalling including:
 
- F1SetupRequest

- F1SetupResponse
 
## ZMQ Configuration
 
The DU uses the ZMQ radio driver for virtual RF operation.
 
ZMQ ports:
 
- TX: 2000

- RX: 2001
 
Configuration:
 
configs/du_zmq.yml
 
## Configuration Files
 
- configs/cu.yml

- configs/du_zmq.yml

- configs/amf.yaml

- configs/open5gs-ue.example.yaml
 
The UE configuration is provided only as an example.

Authentication credentials are intentionally redacted.
 
## Packet Capture
 
The F1-C traffic was captured on the loopback interface using tcpdump.
 
The capture contains SCTP association and F1AP traffic between the CU and DU.
 
## Current Limitation
 
The current testbed successfully demonstrates the CU-DU F1-C

connection and F1AP setup signalling.
 
Full UE-context signalling was not captured in the current run because

the UERANSIM UE did not successfully complete cell selection with the

current ZMQ-based DU setup.
 
