# Open5GS EPC & srsRAN 4G with ZeroMQ UE / RAN Sample Configuration
srsRAN 4G software suite includes a virtual radio which uses the ZeroMQ networking library to transfer radio samples between applications.
Therefore, I used this function to build a simulation environment for the Open5GS CUPS-enabled EPC mobile network.
This configuration is very useful when verifying EPC functionality.
This briefly describes the overall and configuration files in the Proxmox VE environment.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS EPC and srsRAN 4G ZMQ UE / RAN](#changes)
  - [Changes in configuration files of Open5GS EPC C-Plane](#changes_cp)
  - [Changes in configuration files of Open5GS EPC U-Plane](#changes_up)
  - [Changes in configuration files of srsRAN 4G ZMQ UE / RAN](#changes_srs)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE](#changes_ue)
- [Network settings of Open5GS EPC](#network_settings)
  - [Network settings of Open5GS EPC C-Plane](#network_settings_cp)
  - [Network settings of Open5GS EPC U-Plane](#network_settings_up)
- [Build Open5GS and srsRAN 4G ZMQ UE / RAN](#build)
- [Run Open5GS EPC and srsRAN 4G ZMQ UE / RAN](#run)
  - [Run Open5GS EPC C-Plane](#run_cp)
  - [Run Open5GS EPC U-Plane](#run_up)
  - [Run srsRAN 4G ZMQ RAN](#run_ran)
  - [Run srsRAN 4G ZMQ UE](#run_ue)
- [Ping google.com](#ping)
  - [Case for going through PDN 10.45.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)
---

<a id="overview"></a>

## Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network

I created a CUPS-enabled EPC mobile network (Internet reachable) for simulation with the aim of creating an environment in which packets can be sent end-to-end with one PDN for one APN.

The following minimum configuration was set as a condition.
- Only one each for C-Plane, U-Plane and UE.

The built simulation environment is as follows.
**According to [this](https://docs.srsran.com/projects/4g/en/latest/app_notes/source/zeromq/source/index.html#known-issues), srsRAN 4G ZMQ supports only one eNodeB and one UE, so I have confirmed the operation with the following configuration.**

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The EPC / UE / RAN used are as follows.
- EPC - Open5GS v2.7.2 (2024.10.18) - https://github.com/open5gs/open5gs
- UE / RAN - srsRAN 4G (2024.02.01) - https://github.com/srsran/srsRAN_4G

Each VMs are as follows.  
| VM # | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS EPC C-Plane | 192.168.0.111/24 <br> 192.168.0.112/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM2 | Open5GS EPC U-Plane  | 192.168.0.113/24 <br> 192.168.0.114/24 | Ubuntu 24.04 | 1 | 1GB | 20GB |
| VM3 | srsRAN 4G ZMQ RAN (eNodeB) | 192.168.0.121/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |
| VM4 | srsRAN 4G ZMQ UE | 192.168.0.122/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |

Subscriber Information (other information is the same) is as follows.  
| UE | IMSI | APN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000100 | internet | OPc |

I registered these information with the Open5GS WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

PDN is as follows.
| PDN | TUNnel interface of PDN | APN | TUNnel interface of UE |
| --- | --- | --- | --- |
| 10.45.0.0/16 | ogstun | internet | tun_srsue |

The main information of eNodeB is as follows.
| MCC | MNC | TAC | eNodeB ID | Cell ID | E-UTRAN Cell ID |
| --- | --- | --- | --- | --- | --- |
| 001 | 01 | 1 | 0x19b | 0x01 | 0x19b01 |

Additional information.

Open5GS EPC U-Plane worked fine on Raspberry Pi 4 Model B. I used [Ubuntu 20.04 (64bit) for Raspberry Pi 4](https://ubuntu.com/download/raspberry-pi) as the OS. I think it would be convenient to place a compact U-Plane in the edge environment and use it as an end-point for PDN.

In addition, I have not confirmed the communication performance.

<a id="changes"></a>

## Changes in configuration files of Open5GS EPC and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS and srsRAN 4G ZMQ UE / RAN respectively.
- Open5GS v2.7.2 (2024.10.18) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS EPC C-Plane

The following parameters can be used in the logic that selects SGW-U and UPF(PGW-U) as the connection destination by PFCP.

- APN
- TAC (Tracking Area Code)
- e_CellID

For the sake of simplicity, I used only APN this time.

- `open5gs/install/etc/open5gs/mme.yaml`
```diff
--- mme.yaml.orig       2024-10-27 08:48:52.000000000 +0900
+++ mme.yaml    2024-11-09 00:50:59.006621937 +0900
@@ -12,7 +12,7 @@
   freeDiameter: /root/open5gs/install/etc/freeDiameter/mme.conf
   s1ap:
     server:
-      - address: 127.0.0.2
+      - address: 192.168.0.111
   gtpc:
     server:
       - address: 127.0.0.2
@@ -27,14 +27,14 @@
         port: 9090
   gummei:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       mme_gid: 2
       mme_code: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   security:
     integrity_order : [ EIA2, EIA1, EIA0 ]
```
- `open5gs/install/etc/open5gs/sgwc.yaml`
```diff
--- sgwc.yaml.orig      2024-05-02 19:52:00.000000000 +0900
+++ sgwc.yaml   2024-05-03 20:30:16.000000000 +0900
@@ -14,10 +14,11 @@
       - address: 127.0.0.3
   pfcp:
     server:
-      - address: 127.0.0.3
+      - address: 192.168.0.111
     client:
       sgwu:
-        - address: 127.0.0.6
+        - address: 192.168.0.113
+          apn: internet
 
 ################################################################################
 # GTP-C Server
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2024-05-02 19:52:00.000000000 +0900
+++ smf.yaml    2024-05-03 20:32:04.000000000 +0900
@@ -9,27 +9,19 @@
 #    peer: 64
 
 smf:
-  sbi:
-    server:
-      - address: 127.0.0.4
-        port: 7777
-    client:
-#      nrf:
-#        - uri: http://127.0.0.10:7777
-      scp:
-        - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.112
     client:
       upf:
-        - address: 127.0.0.7
+        - address: 192.168.0.114
+          dnn: internet
   gtpc:
     server:
       - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.0.112
   metrics:
     server:
       - address: 127.0.0.4
@@ -37,13 +29,10 @@
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
   dns:
     - 8.8.8.8
     - 8.8.4.4
-    - 2001:4860:4860::8888
-    - 2001:4860:4860::8844
   mtu: 1400
 #  p-cscf:
 #    - 127.0.0.1
```

<a id="changes_up"></a>

### Changes in configuration files of Open5GS EPC U-Plane

- `open5gs/install/etc/open5gs/sgwu.yaml`
```diff
--- sgwu.yaml.orig      2024-05-02 19:52:00.000000000 +0900
+++ sgwu.yaml   2024-05-03 20:34:28.000000000 +0900
@@ -11,13 +11,13 @@
 sgwu:
   pfcp:
     server:
-      - address: 127.0.0.6
+      - address: 192.168.0.113
     client:
 #      sgwc:    # SGW-U PFCP Client try to associate SGW-C PFCP Server
 #        - address: 127.0.0.3
   gtpu:
     server:
-      - address: 127.0.0.6
+      - address: 192.168.0.113
 
 ################################################################################
 # PFCP Server
```
- `open5gs/install/etc/open5gs/upf.yaml`
```diff
--- upf.yaml.orig       2024-05-02 19:52:00.000000000 +0900
+++ upf.yaml    2024-05-03 20:36:00.000000000 +0900
@@ -11,18 +11,18 @@
 upf:
   pfcp:
     server:
-      - address: 127.0.0.7
+      - address: 192.168.0.114
     client:
 #      smf:     #  UPF PFCP Client try to associate SMF PFCP Server
 #        - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.7
+      - address: 192.168.0.114
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
+      dev: ogstun
   metrics:
     server:
       - address: 127.0.0.7
```

<a id="changes_srs"></a>

### Changes in configuration files of srsRAN 4G ZMQ UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `srsRAN_4G/build/srsenb/enb.conf`
```diff
--- enb.conf.example    2024-02-03 23:26:02.000000000 +0900
+++ enb.conf    2024-03-29 20:18:47.500592921 +0900
@@ -22,9 +22,9 @@
 enb_id = 0x19B
 mcc = 001
 mnc = 01
-mme_addr = 127.0.1.100
-gtp_bind_addr = 127.0.1.1
-s1c_bind_addr = 127.0.1.1
+mme_addr = 192.168.0.111
+gtp_bind_addr = 192.168.0.121
+s1c_bind_addr = 192.168.0.121
 s1c_bind_port = 0
 n_prb = 50
 #tm = 4
@@ -80,8 +80,8 @@
 #time_adv_nsamples = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = fail_on_disconnect=true,tx_port=tcp://*:2000,rx_port=tcp://localhost:2001,id=enb,base_srate=23.04e6
+device_name = zmq
+device_args = fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
 
 #####################################################################
 # Packet capture configuration
```
- `srsRAN_4G/build/srsenb/rr.conf`
```diff
--- rr.conf.example     2024-02-03 23:26:02.000000000 +0900
+++ rr.conf     2023-05-02 11:52:54.000000000 +0900
@@ -55,7 +55,7 @@
   {
     // rf_port = 0;
     cell_id = 0x01;
-    tac = 0x0007;
+    tac = 0x0001;
     pci = 1;
     // root_seq_idx = 204;
     dl_earfcn = 3350;
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE

- `srsRAN_4G/build/srsue/ue.conf`
```diff
--- ue.conf.example     2024-02-03 23:26:02.000000000 +0900
+++ ue.conf     2024-03-29 20:22:16.358858947 +0900
@@ -42,8 +42,8 @@
 #continuous_tx     = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = tx_port=tcp://*:2001,rx_port=tcp://localhost:2000,id=ue,base_srate=23.04e6
+device_name = zmq
+device_args = tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
 
 #####################################################################
 # EUTRA RAT configuration
@@ -139,9 +139,9 @@
 [usim]
 mode = soft
 algo = milenage
-opc  = 63BFA50EE6523365FF14C1F45F88737D
-k    = 00112233445566778899aabbccddeeff
-imsi = 001010123456780
+opc  = E8ED289DEBA952E4283B54E88E6183CA
+k    = 465B5CE8B199B49FAA5F0A2EE238A6BC
+imsi = 001010000000100
 imei = 353490069873319
 #reader =
 #pin  = 1234
@@ -180,8 +180,8 @@
 #                      Supported: 0 - NULL, 1 - Snow3G, 2 - AES, 3 - ZUC
 #####################################################################
 [nas]
-#apn = internetinternet
-#apn_protocol = ipv4
+apn = internet
+apn_protocol = ipv4
 #user = srsuser
 #pass = srspass
 #force_imsi_attach = false
```

<a id="network_settings"></a>

## Network settings of Open5GS EPC

<a id="network_settings_cp"></a>

### Network settings of Open5GS EPC C-Plane

Add IP address for SMF(PGW-C).
```
ip addr add 192.168.0.112/24 dev ens19
```
**Note. `ens19` is the network interface of `192.168.0.0/24` in my Proxmox VE environment.
Please change it according to your environment.**

<a id="network_settings_up"></a>

### Network settings of Open5GS EPC U-Plane

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, add IP address for UPF(PGW-U) and configure the TUNnel interface and NAPT.
```
ip addr add 192.168.0.114/24 dev ens19

ip tuntap add name ogstun mode tun
ip addr add 10.45.0.1/16 dev ogstun
ip link set ogstun up

iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```

<a id="build"></a>

## Build Open5GS and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS and srsRAN 4G ZMQ UE / RAN respectively.
- Open5GS v2.7.2 (2024.10.18) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- srsRAN 4G (2024.02.01) - https://github.com/s5uishida/build_srsran_4g_zmq_disable_rf_plugins

Install MongoDB on Open5GS EPC C-Plane machine.
It is not necessary to install MongoDB on Open5GS EPC U-Plane machines.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS EPC and srsRAN 4G ZMQ UE / RAN

First run the EPC, then the RAN, and the UE.

<a id="run_cp"></a>

### Run Open5GS EPC C-Plane

First, run Open5GS EPC C-Plane.

- Open5GS EPC C-Plane
```
./install/bin/open5gs-mmed &
./install/bin/open5gs-sgwcd &
./install/bin/open5gs-smfd &
./install/bin/open5gs-hssd &
./install/bin/open5gs-pcrfd &
```

<a id="run_up"></a>

### Run Open5GS EPC U-Plane

Next, run Open5GS EPC U-Plane.

- Open5GS EPC U-Plane
```
./install/bin/open5gs-sgwud &
./install/bin/open5gs-upfd &
```

<a id="run_ran"></a>

### Run srsRAN 4G ZMQ RAN

Run srsRAN 4G ZMQ RAN and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsenb
# ./src/srsenb enb.conf
---  Software Radio Systems LTE eNodeB  ---

Reading configuration file enb.conf...

Built in Release mode using commit ec29b0c1f on branch master.

Opening 1 channels in RF device=zmq with args=fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=enb
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.122:2001
CH0 tx_port=tcp://192.168.0.121:2000
CH0 fail_on_disconnect=true
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Setting frequency: DL=2680.0 Mhz, UL=2560.0 MHz for cc_idx=0 nof_prb=50

==== eNodeB started ===
Type <t> to view trace
```
The Open5GS C-Plane log when executed is as follows.
```
11/09 01:24:56.954: [mme] INFO: eNB-S1 accepted[192.168.0.121]:48452 in s1_path module (../src/mme/s1ap-sctp.c:114)
11/09 01:24:56.954: [mme] INFO: eNB-S1 accepted[192.168.0.121] in master_sm module (../src/mme/mme-sm.c:108)
11/09 01:24:56.954: [mme] INFO: [Added] Number of eNBs is now 1 (../src/mme/mme-context.c:2981)
11/09 01:24:56.954: [mme] INFO: eNB-S1[192.168.0.121] max_num_of_ostreams : 30 (../src/mme/mme-sm.c:157)
```

<a id="run_ue"></a>

### Run srsRAN 4G ZMQ UE

Run srsRAN 4G ZMQ UE and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsue
# ./src/srsue ue.conf
Reading configuration file ue.conf...

Built in Release mode using commit ec29b0c1f on branch master.

Opening 1 channels in RF device=zmq with args=tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=ue
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.121:2000
CH0 tx_port=tcp://192.168.0.122:2001
Waiting PHY to initialize ... done!
Attaching UE...
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
.
Found Cell:  Mode=FDD, PCI=1, PRB=50, Ports=1, CP=Normal, CFO=-0.2 KHz
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Found PLMN:  Id=00101, TAC=1
Random Access Transmission: seq=10, tti=181, ra-rnti=0x2
RRC Connected
Random Access Complete.     c-rnti=0x46, ta=0
Network attach successful. IP: 10.45.0.2
 nTp) ((t) 8/11/2024 16:26:28 TZ:99
```
The Open5GS C-Plane log when executed is as follows.
```
11/09 01:26:28.578: [mme] INFO: InitialUEMessage (../src/mme/s1ap-handler.c:426)
11/09 01:26:28.579: [mme] INFO: [Added] Number of eNB-UEs is now 1 (../src/mme/mme-context.c:4952)
11/09 01:26:28.579: [mme] INFO: Unknown UE by S_TMSI[G:2,C:1,M_TMSI:0xc0000607] (../src/mme/s1ap-handler.c:505)
11/09 01:26:28.579: [mme] INFO:     ENB_UE_S1AP_ID[1] MME_UE_S1AP_ID[1] TAC[1] CellID[0x19b01] (../src/mme/s1ap-handler.c:602)
11/09 01:26:28.579: [mme] INFO: Unknown UE by GUTI[G:2,C:1,M_TMSI:0xc0000607] (../src/mme/mme-context.c:3751)
11/09 01:26:28.579: [mme] INFO: [Added] Number of MME-UEs is now 1 (../src/mme/mme-context.c:3543)
11/09 01:26:28.579: [emm] INFO: [] Attach request (../src/mme/emm-sm.c:438)
11/09 01:26:28.579: [emm] INFO:     GUTI[G:2,C:1,M_TMSI:0xc0000607] IMSI[Unknown IMSI] (../src/mme/emm-handler.c:232)
11/09 01:26:28.600: [emm] INFO: Identity response (../src/mme/emm-sm.c:408)
11/09 01:26:28.600: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:424)
11/09 01:26:28.667: [mme] INFO: [Added] Number of MME-Sessions is now 1 (../src/mme/mme-context.c:4966)
11/09 01:26:28.708: [sgwc] INFO: [Added] Number of SGWC-UEs is now 1 (../src/sgwc/context.c:239)
11/09 01:26:28.708: [sgwc] INFO: [Added] Number of SGWC-Sessions is now 1 (../src/sgwc/context.c:905)
11/09 01:26:28.709: [sgwc] INFO: UE IMSI[001010000000100] APN[internet] (../src/sgwc/s11-handler.c:268)
11/09 01:26:28.709: [gtp] INFO: gtp_connect() [127.0.0.4]:2123 (../lib/gtp/path.c:60)
11/09 01:26:28.710: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1031)
11/09 01:26:28.710: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3119)
11/09 01:26:28.710: [smf] INFO: UE IMSI[001010000000100] APN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/s5c-handler.c:303)
11/09 01:26:28.713: [gtp] INFO: gtp_connect() [192.168.0.114]:2152 (../lib/gtp/path.c:60)
11/09 01:26:28.974: [emm] INFO: [001010000000100] Attach complete (../src/mme/emm-sm.c:1451)
11/09 01:26:28.975: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:272)
11/09 01:26:28.975: [emm] INFO:     UTC [2024-11-08T16:26:28] Timezone[0]/DST[0] (../src/mme/emm-handler.c:278)
11/09 01:26:28.975: [emm] INFO:     LOCAL [2024-11-09T01:26:28] Timezone[32400]/DST[0] (../src/mme/emm-handler.c:282)
```
The Open5GS U-Plane log when executed is as follows.
```
11/09 01:26:28.713: [sgwu] INFO: UE F-SEID[UP:0x55 CP:0x92e] (../src/sgwu/context.c:170)
11/09 01:26:28.713: [sgwu] INFO: [Added] Number of SGWU-Sessions is now 1 (../src/sgwu/context.c:175)
11/09 01:26:28.717: [upf] INFO: [Added] Number of UPF-Sessions is now 1 (../src/upf/context.c:209)
11/09 01:26:28.717: [gtp] INFO: gtp_connect() [192.168.0.113]:2152 (../lib/gtp/path.c:60)
11/09 01:26:28.717: [gtp] INFO: gtp_connect() [192.168.0.112]:2152 (../lib/gtp/path.c:60)
11/09 01:26:28.717: [upf] INFO: UE F-SEID[UP:0x1d9 CP:0xac8] APN[internet] PDN-Type[1] IPv4[10.45.0.2] IPv6[] (../src/upf/context.c:495)
11/09 01:26:28.717: [upf] INFO: UE F-SEID[UP:0x1d9 CP:0xac8] APN[internet] PDN-Type[1] IPv4[10.45.0.2] IPv6[] (../src/upf/context.c:495)
11/09 01:26:28.718: [gtp] INFO: gtp_connect() [192.168.0.114]:2152 (../lib/gtp/path.c:60)
11/09 01:26:28.979: [gtp] INFO: gtp_connect() [192.168.0.121]:2152 (../lib/gtp/path.c:60)
```
The result of `ip addr show` on VM4 (UE) is as follows.
```
# ip addr show
...
8: tun_srsue: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global tun_srsue
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the TUN interface on VM4 (UE) and try `ping`.

<a id="ping_1"></a>

### Case for going through PDN 10.45.0.0/16

Execute `tcpdump` on VM2 (U-Plane) and check that the packet goes through `if=ogstun`.
- `ping google.com` on VM4 (UE)
```
# ping google.com -I tun_srsue -n
PING google.com (142.250.207.14) from 10.45.0.2 tun_srsue: 56(84) bytes of data.
64 bytes from 142.250.207.14: icmp_seq=1 ttl=111 time=56.9 ms
64 bytes from 142.250.207.14: icmp_seq=2 ttl=111 time=51.1 ms
64 bytes from 142.250.207.14: icmp_seq=3 ttl=111 time=46.7 ms
```
- Run `tcpdump` on VM2 (U-Plane)
```
# tcpdump -i ogstun -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ogstun, link-type RAW (Raw IP), snapshot length 262144 bytes
01:29:40.910538 IP 10.45.0.2 > 142.250.207.14: ICMP echo request, id 6, seq 1, length 64
01:29:40.927749 IP 142.250.207.14 > 10.45.0.2: ICMP echo reply, id 6, seq 1, length 64
01:29:41.907018 IP 10.45.0.2 > 142.250.207.14: ICMP echo request, id 6, seq 2, length 64
01:29:41.923542 IP 142.250.207.14 > 10.45.0.2: ICMP echo reply, id 6, seq 2, length 64
01:29:42.903451 IP 10.45.0.2 > 142.250.207.14: ICMP echo request, id 6, seq 3, length 64
01:29:42.920091 IP 142.250.207.14 > 10.45.0.2: ICMP echo reply, id 6, seq 3, length 64
```
In addition to `ping`, you may try to access the web by specifying the TUNnel interface with `curl` as follows.
- `curl google.com` on VM4 (UE)
```
# curl --interface tun_srsue google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM2 (U-Plane)
```
01:30:21.134561 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [S], seq 318643480, win 64240, options [mss 1460,sackOK,TS val 2103061751 ecr 0,nop,wscale 7], length 0
01:30:21.149836 IP 142.250.207.14.80 > 10.45.0.2.43110: Flags [S.], seq 1160569541, ack 318643481, win 65535, options [mss 1412,sackOK,TS val 1973624124 ecr 2103061751,nop,wscale 8], length 0
01:30:21.264694 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 2103062075 ecr 1973624124], length 0
01:30:21.264760 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [P.], seq 1:75, ack 1, win 502, options [nop,nop,TS val 2103062075 ecr 1973624124], length 74: HTTP: GET / HTTP/1.1
01:30:21.281938 IP 142.250.207.14.80 > 10.45.0.2.43110: Flags [.], ack 75, win 1050, options [nop,nop,TS val 1973624256 ecr 2103062075], length 0
01:30:21.317934 IP 142.250.207.14.80 > 10.45.0.2.43110: Flags [P.], seq 1:774, ack 75, win 1050, options [nop,nop,TS val 1973624292 ecr 2103062075], length 773: HTTP: HTTP/1.1 301 Moved Permanently
01:30:21.331529 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [.], ack 774, win 501, options [nop,nop,TS val 2103062158 ecr 1973624292], length 0
01:30:21.331562 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [F.], seq 75, ack 774, win 501, options [nop,nop,TS val 2103062158 ecr 1973624292], length 0
01:30:21.347378 IP 142.250.207.14.80 > 10.45.0.2.43110: Flags [F.], seq 774, ack 76, win 1050, options [nop,nop,TS val 1973624321 ecr 2103062158], length 0
01:30:21.374564 IP 10.45.0.2.43110 > 142.250.207.14.80: Flags [.], ack 775, win 501, options [nop,nop,TS val 2103062186 ecr 1973624321], length 0
```
You could now create the end-to-end TUN interface on the PDN and send any packets on the network.

---
In investigating private LTE, I have built a simulation environment and can now use a very useful system for investigating CUPS-enabled EPC and MEC of LTE mobile network. I would like to thank the excellent developers and all the contributors of Open5GS and srsRAN 4G.

<a id="changelog"></a>

## Changelog (summary)

- [2024.11.08] Updated to Open5GS v2.7.2 (2024.10.18).
- [2024.03.29] Updated to Open5GS v2.7.0 (2024.03.24).
- [2023.05.02] Initial release.
