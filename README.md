# SDN Multi-Switch Flow Table Analyzer
### Using Ryu Controller + Mininet + OpenFlow 1.3

---

## Project Overview

This project demonstrates **Software Defined Networking (SDN)** behavior across a multi-switch topology. A centralized Ryu controller dynamically analyzes incoming traffic and installs forwarding rules on OpenFlow-enabled switches in real time.

The core idea: instead of switches making their own routing decisions, **all intelligence lives in the controller**. Switches are just dumb forwarders — they ask the controller what to do when they see an unknown packet.

The network is tested under three conditions:
-  Normal connectivity
-  Link failure (s2–s3 goes down)
-  Link recovery (s2–s3 comes back up)

---

## Network Topology

```
   h1          h2          h3          h4
    |            |           |           |
   s1 --------- s2 --------- s3 -------- s4
```

| Component | Details |
|---|---|
| Switches | s1, s2, s3, s4 (OpenFlow 1.3, OVS) |
| Hosts | h1, h2, h3, h4 |
| Controller | Ryu (flowanalyser.py) |
| Protocol | OpenFlow 1.3 |
| Topology | Linear (4 switches) |

---

## Requirements

- Ubuntu 20.04 / 22.04 (bare metal or VM)
- Python 3.8–3.10
- Mininet
- Open vSwitch (OVS)
- Ryu Controller
- Wireshark (optional, for packet analysis)
- iperf (optional, for throughput testing)

---

## Installation

```bash
# System packages
sudo apt update
sudo apt install python3-pip mininet openvswitch-switch wireshark -y

# Python packages
pip3 install ryu eventlet==0.33.3 tinyrpc==0.9.4

# Fix eventlet/Python 3.10 timeout compatibility issue
sed -i 's/base.is_timeout = property(lambda _: True)/pass  # patched/' \
  ~/.local/lib/python3.10/site-packages/eventlet/timeout.py
```

---

## Project Structure

```
SDN_merwyn_multi_flow/
├── controller/
│   └── flowanalyser.py       # Ryu L2 learning switch controller
└── README.md
```

---

## How to Run

### Step 1 — Start the Ryu Controller (Terminal 1)

```bash
# Clean any leftover Mininet state
sudo mn -c

cd ~/Storage/SDN_merwyn_multi_flow/controller
ryu-manager ./flowanalyser.py
```

Expected output:
```
loading app ./flowanalyser.py
instantiating app ./flowanalyser.py of FlowAnalyzer
instantiating app ryu.controller.ofp_handler of OFPHandler
```

---

### Step 2 — Launch Mininet Topology (Terminal 2)

```bash
sudo mn --topo linear,4 --mac --switch ovsk --controller remote,ip=127.0.0.1,port=6633
```

Expected:
```
*** Starting CLI:
mininet>
```

---

### Step 3 — Test Normal Connectivity

```
mininet> pingall
```

Expected:
```
*** Results: 0% dropped (12/12 received)
```

---

### Step 4 — Verify Flow Tables (Terminal 3)

```bash
sudo ovs-ofctl dump-flows s1
sudo ovs-ofctl dump-flows s2
sudo ovs-ofctl dump-flows s3
sudo ovs-ofctl dump-flows s4
```

Or from inside Mininet:
```
mininet> sh ovs-ofctl dump-flows s1
```

Expected output (per switch):
```
priority=1, in_port="s1-eth1", dl_dst=00:00:00:00:00:02  actions=output:"s1-eth2"
priority=0  actions=CONTROLLER:65535
```

Two rule types:
- `priority=1` — learned MAC→port forwarding rules (installed by controller)
- `priority=0` — table-miss rule (unknown packets go to controller)

---

### Step 5 — Link Failure Test

```
mininet> py net.configLinkStatus('s2','s3','down')
mininet> pingall
```

Expected:
- h3 and h4 become **unreachable** from h1/h2
- Partial packet loss in pingall output

---

### Step 6 — Link Recovery

```
mininet> py net.configLinkStatus('s2','s3','up')
mininet> pingall
```

Expected:
```
0% dropped — all hosts reachable again
```

---

### Step 7 — Wireshark OpenFlow Capture (Optional)

```bash
sudo wireshark
```

1. Select interface: **`lo`** (loopback)
2. Apply display filter: `openflow_v4`
3. In Mininet, run: `h1 ping -c 5 h4`

What to look for:

| Message | Meaning |
|---|---|
| `OFPT_PACKET_IN` | Switch sends unknown packet to controller |
| `OFPT_FLOW_MOD` | Controller pushes a new flow rule to switch |
| `OFPT_PACKET_OUT` | Controller tells switch to forward the packet |
| `OFPT_ECHO_REQUEST/REPLY` | Keepalive heartbeat between switch and controller |

First ping triggers a burst of PACKET_IN → FLOW_MOD. Subsequent pings are handled directly by switch rules (no controller involvement).

---

### Step 8 — Throughput Test with iperf (Optional)

**Normal:**
```
mininet> h1 iperf -s &
mininet> h4 iperf -c h1
```

**After failure:**
```
mininet> py net.configLinkStatus('s2','s3','down')
mininet> h4 iperf -c h1
```
Expected: connection fails or significant drop

**After recovery:**
```
mininet> py net.configLinkStatus('s2','s3','up')
mininet> h4 iperf -c h1
```
Expected: throughput restored

---

## Performance Summary

| Scenario | Packet Loss | Throughput |
|---|---|---|
| Normal | 0% | High |
| Link Failure | Partial (h3/h4 unreachable) | Fails |
| Link Recovery | 0% | Restored |

---

## How the Controller Works

`flowanalyser.py` implements a **reactive L2 learning switch**:

1. On switch connect → installs a default table-miss flow rule (send everything to controller)
2. On `PACKET_IN` → reads source MAC and learns which port it came from (`mac_to_port` table)
3. If destination MAC is known → installs a specific flow rule and forwards
4. If destination MAC is unknown → floods out all ports
5. Next time same flow arrives → switch handles it **without involving the controller**

This is the fundamental SDN reactive model — **no pre-configured routing, everything learned from live traffic**.

---

## Key Concepts

| Concept | Explanation |
|---|---|
| **Reactive SDN** | Flow rules installed on demand, triggered by traffic |
| **Table-miss rule** | Priority 0 catch-all — unknown packets go to controller |
| **Flow installation** | Controller pushes `OFP_FLOW_MOD` to switch after learning |
| **MAC learning** | Per-switch `dpid → {mac: port}` table maintained in controller |
| **OpenFlow 1.3** | Protocol used between Ryu controller and OVS switches |

---

## Conclusion

This project demonstrates:

- **Centralized SDN control** — one Ryu controller manages all 4 switches
- **Dynamic flow installation** — rules are reactively pushed based on live traffic, not pre-configured
- **MAC learning across multiple switches** — controller maintains per-switch forwarding tables
- **Resilience testing** — network responds to link failure and recovery without manual intervention
- **OpenFlow observability** — flow tables and Wireshark captures provide full visibility into SDN behavior

The experiment confirms that SDN enables programmable, observable, and adaptive networking — where the control plane is fully separated from the data plane and can be modified in software at runtime.

---

*PES University | Electronics & Communication | Computer Networks Lab*
