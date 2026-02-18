# Bridge Network with Multi-Interface NAT — Architecture

## Network Diagram

```
                         EXTERNAL CLIENTS
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
      ┌────────────┐ ┌────────────┐ ┌────────────┐
      │  Ethernet  │ │    WiFi    │ │  5G Modem  │
      │    eth0    │ │   wlan0    │ │   wwan0    │
      │ 192.168.x  │ │   10.0.x   │ │  carrier   │
      └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
            │              │              │
            └──────────────┼──────────────┘
                           │
  ╔════════════════════════════════════════════════════════╗
  ║               HOST (EVE-OS + k3s node)                 ║
  ║                                                        ║
  ║  ┌─────────────────────────────────────────────────┐   ║
  ║  │            iptables NAT (nat table)             │   ║
  ║  │                                                 │   ║
  ║  │  PREROUTING chain:                              │   ║
  ║  │  ┌───────────────────────────────────────────┐  │   ║
  ║  │  │ DNAT 0.0.0.0:2376  → 10.20.0.10:2376      │  │   ║
  ║  │  │ DNAT 0.0.0.0:30080 → 10.20.0.10:8080      │  │   ║
  ║  │  └───────────────────────────────────────────┘  │   ║
  ║  │                                                 │   ║
  ║  │  POSTROUTING chain:                             │   ║
  ║  │  ┌───────────────────────────────────────────┐  │   ║
  ║  │  │ MASQUERADE src 10.20.0.0/24               │  │   ║
  ║  │  │ (outbound from pods → any interface)      │  │   ║
  ║  │  └───────────────────────────────────────────┘  │   ║
  ║  │                                                 │   ║
  ║  │  FORWARD chain:                                 │   ║
  ║  │  ┌───────────────────────────────────────────┐  │   ║
  ║  │  │ ACCEPT -i dind-br0                        │  │   ║
  ║  │  │ ACCEPT -o dind-br0 RELATED,ESTABLISHED    │  │   ║
  ║  │  │ ACCEPT -d 10.20.0.10 --dport 2376         │  │   ║
  ║  │  │ ACCEPT -d 10.20.0.10 --dport 8080         │  │   ║
  ║  │  └───────────────────────────────────────────┘  │   ║
  ║  └─────────────────────────────────────────────────┘   ║
  ║                          │                             ║
  ║                          │ L3 routing                  ║
  ║                          ▼                             ║
  ║              ┌───────────────────┐                     ║
  ║              │     dind-br0      │                     ║
  ║              │   10.20.0.1/24    │                     ║
  ║              │   (linux bridge)  │                     ║
  ║              └─────┬───────┬─────┘                     ║
  ║                    │       │                           ║
  ║                veth pair   veth pair                   ║
  ║                    │       │                           ║
  ╚════════════════════════════════════════════════════════╝
                       │       │
                       ▼       ▼
  ┌───────────────────────┐ ┌───────────────────────┐
  │       DinD Pod        │ │      Future Pod       │
  │                       │ │                       │
  │  eth0: flannel IP     │ │  eth0: flannel IP     │
  │  net1: 10.20.0.10     │ │  net1: 10.20.0.11     │
  │                       │ │                       │
  │  ┌─────────────────┐  │ │  ┌─────────────────┐  │
  │  │  dockerd        │  │ │  │  your app       │  │
  │  │  :2376 (TLS)    │  │ │  │  :8080          │  │
  │  │  containers     │  │ │  │                 │  │
  │  │  inside DinD    │  │ │  └─────────────────┘  │
  │  └─────────────────┘  │ │                       │
  └───────────────────────┘ └───────────────────────┘
```

## Kubernetes Resources Deployed

```
  ┌───────────────────────────────────────────────────────────┐
  │                       Helm Release                        │
  │                                                           │
  │  ┌─────────────────────────────────┐                      │
  │  │  NetworkAttachmentDefinition    │  Tells Multus to use │
  │  │  <release>-bridge               │  bridge CNI plugin   │
  │  │                                 │  with host-local     │
  │  │  bridge: dind-br0               │  IPAM                │
  │  │  subnet: 10.20.0.0/24           │                      │
  │  └─────────────────────────────────┘                      │
  │                                                           │
  │  ┌─────────────────────────────────┐                      │
  │  │  DaemonSet (bridge-nat)         │  Runs on every node  │
  │  │  ┌───────────────────────────┐  │                      │
  │  │  │  ConfigMap: setup-nat.sh  │  │  hostNetwork: true   │
  │  │  │  - sysctl ip_forward=1    │  │  privileged: true    │
  │  │  │  - iptables MASQUERADE    │  │                      │
  │  │  │  - iptables DNAT rules    │  │  Cleans up rules on  │
  │  │  │  - iptables FORWARD       │  │  SIGTERM             │
  │  │  │  - trap SIGTERM cleanup   │  │                      │
  │  │  └───────────────────────────┘  │                      │
  │  └─────────────────────────────────┘                      │
  │                                                           │
  │  ┌─────────────────────────────────┐                      │
  │  │  Deployment (dind)              │                      │
  │  │                                 │                      │
  │  │  annotation:                    │  Multus attaches pod │
  │  │    k8s.v1.cni.cncf.io/networks  │  to dind-br0 bridge  │
  │  │                                 │                      │
  │  │  initContainer:                 │  Waits for net1 IP   │
  │  │    wait-for-bridge              │  before starting     │
  │  │                                 │                      │
  │  │  containers:                    │                      │
  │  │    - dind (dockerd)             │                      │
  │  │    - gc (garbage collector)     │                      │
  │  └─────────────────────────────────┘                      │
  │                                                           │
  │  ┌─────────────────────────────────┐                      │
  │  │  Service (ClusterIP)            │  In-cluster access   │
  │  │  :2376 -> dind:2376             │  still works via     │
  │  │  :8080 -> dind:8080             │  normal k8s service  │
  │  └─────────────────────────────────┘                      │
  └───────────────────────────────────────────────────────────┘
```

## Traffic Flows

### Inbound: External client reaches a pod

```
  Client (any network)
    │
    │  TCP SYN to <host-ip>:30080
    ▼
  ┌──────────────────────────────────────┐
  │ eth0 / wlan0 / wwan0                 │
  │ Packet arrives on any interface      │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │ PREROUTING (nat table)               │
  │                                      │
  │ Match:  --dport 30080                │
  │ Action: DNAT -> 10.20.0.10:8080      │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │ FORWARD chain                        │
  │                                      │
  │ Match:  -d 10.20.0.10 --dport 8080   │
  │ Action: ACCEPT                       │
  └──────────────────┬───────────────────┘
                     │
                     ▼  routed to dind-br0
  ┌──────────────────────────────────────┐
  │ dind-br0 -> veth -> Pod              │
  │                                      │
  │ Pod receives packet on               │
  │ net1 (10.20.0.10:8080)               │
  └──────────────────────────────────────┘
```

### Outbound: Pod reaches the internet

```
  Pod (10.20.0.10)
    │
    │  e.g. curl https://registry.hub.docker.com
    ▼
  ┌──────────────────────────────────────┐
  │ dind-br0 (gateway)                   │
  │ 10.20.0.1                            │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │ FORWARD chain                        │
  │                                      │
  │ Match:  -i dind-br0                  │
  │ Action: ACCEPT                       │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │ POSTROUTING (nat table)              │
  │                                      │
  │ Match:  -s 10.20.0.0/24              │
  │         ! -o dind-br0                │
  │ Action: MASQUERADE                   │
  │                                      │
  │ Source IP rewritten to the outgoing  │
  │ interface's IP (eth0/wlan0/wwan0)    │
  └──────────────────┬───────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────┐
  │ Routed via kernel's default route    │
  │ to the appropriate physical          │
  │ interface                            │
  └──────────────────────────────────────┘
```

### Per-port asymmetric routing (with `routing:` block)

When a port-forwarding rule has a `.routing` block, inbound connections on that port get a
unique fwmark, which maps to a dedicated routing table whose default route points to the
interface the rule's traffic arrives on. This ensures DNAT replies exit the same interface
the connection came in on — even when different ports arrive on different physical interfaces.

Example: two rules — port 30080 arrives on `eth0` (fwmark 0x65, table 101), port 30443 on `eth1` (fwmark 0x66, table 102).

```
  ┌────────────────────────────────────────────────────────────────┐
  │                      ip rule table (abbreviated)               │
  │                                                                │
  │  priority 100: fwmark 0x65 → table 101 (default via eth0_gw)  │
  │  priority 100: fwmark 0x66 → table 102 (default via eth1_gw)  │
  │  priority 100: fwmark 0x64 → table 100 (global fallback)      │
  │  priority 110: from 10.20.0.0/24 → table 100 (pod outbound)   │
  └────────────────────────────────────────────────────────────────┘

  Inbound: client → host:30080 (eth0)
    mangle/PREROUTING DIND-MARK
      dport=30080, NEW  →  MARK 0x65
                           CONNMARK --save-mark (! --mark 0x0)
    nat/PREROUTING     →  DNAT to 10.20.0.10:8080
    ip rule fwmark 0x65 → table 101 → 10.20.0.0/24 dev dind-br0 → pod

  Reply: pod → client (must exit eth0)
    mangle/PREROUTING DIND-MARK
      ESTABLISHED       →  CONNMARK --restore-mark → mark=0x65
    ip rule fwmark 0x65 → table 101 → default via eth0_gw dev eth0
    nat/POSTROUTING     →  MASQUERADE (src becomes eth0 IP)  ✓

  Pod-originated outbound: pod → 8.8.8.8
    mangle/PREROUTING: no hostport match, mark stays 0
    ip rule from 10.20.0.0/24 → table 100 → default via eth5
    nat/POSTROUTING    →  MASQUERADE (src becomes eth5 IP)
```

## Why This Architecture?

### Problem
WiFi and 5G interfaces **cannot be L2-bridged** with each other or with ethernet. Linux bridging only works between interfaces on the same L2 domain. Wireless and cellular interfaces manage their own L2 associations.

### Solution
The bridge only connects **pods together** at L2. The **host kernel** handles L3 routing between the bridge subnet and all physical interfaces, using:

| Mechanism | Purpose |
|-----------|---------|
| **DNAT on 0.0.0.0** | Captures inbound traffic arriving on *any* interface and redirects to the pod IP |
| **MASQUERADE** | Rewrites pod source IPs to the outgoing interface's IP for outbound traffic |
| **FORWARD ACCEPT** | Allows the kernel to route packets between the bridge and physical interfaces |
| **ip_forward=1** | Enables the kernel's packet forwarding between interfaces |

### Why not host networking?
Host networking (`hostNetwork: true`) would bypass Kubernetes networking entirely. Pods would share the host's IP and port space, creating conflicts and losing network isolation. The bridge approach keeps pods in their own network namespace with dedicated IPs while still being reachable from any external interface.

### Why not NodePort?
NodePort works through kube-proxy, which binds on `0.0.0.0` — but it adds an extra hop and only works for Kubernetes-aware traffic. The bridge + DNAT approach is more direct and gives full control over port mappings independent of the Kubernetes service mesh.

## Configuration Reference

```yaml
bridge:
  enabled: false              # Toggle the entire bridge setup
  name: "dind-br0"            # Linux bridge interface name
  cniVersion: "0.3.1"         # CNI spec version
  isGateway: true             # Bridge gets the gateway IP
  ipMasq: false               # We handle masquerade ourselves
  hairpinMode: true           # Pod-to-self via bridge works
  ipam:
    type: "host-local"        # Node-local IP allocation
    subnet: "10.20.0.0/24"    # Bridge subnet
    rangeStart: "10.20.0.10"  # First pod IP
    rangeEnd: "10.20.0.50"    # Last pod IP
    gateway: "10.20.0.1"      # Bridge interface IP

  # Global (preferred) policy routing — used for pod-originated outbound traffic
  # and as fallback for port-forward rules that do not have a per-port routing block.
  routing:
    enabled: true
    tableId: 100              # Custom routing table ID (1-252)
    tableName: "dind_net1"    # Name registered in /etc/iproute2/rt_tables
    fwmark: "0x64"            # Firewall mark (fallback for rules without .routing)
    rulePriority: 100         # ip rule priority; source-based rule gets priority+10
    outbound:
      interface: "eth5"       # Default outbound interface for pod-originated traffic
      gateway: ""             # Optional explicit gateway; auto-discovered if empty
      useDefaultRoute: false  # Copy default route from main table instead

  portForwarding:
    rules:
      - name: docker-tls      # Human-readable label
        hostPort: 2376         # Port on host (all interfaces)
        destIP: "10.20.0.10"   # Pod IP on bridge
        destPort: 2376         # Port inside pod
        protocol: tcp

      - name: web-app
        hostPort: 30080
        destIP: "10.20.0.10"
        destPort: 8080
        protocol: tcp
        # Per-port isolated routing (optional).
        # When set, replies to connections arriving on this port exit via this
        # rule's dedicated routing table — guaranteeing they leave on the same
        # interface the original packet arrived on (asymmetric multi-interface NAT).
        routing:
          fwmark: "0x65"            # Unique hex mark; must not collide with other rules
          tableId: 101              # Routing table ID (1-252; != bridge.routing.tableId)
          tableName: "dind_eth0"    # Name registered in /etc/iproute2/rt_tables
          outbound:
            interface: "eth0"       # Replies exit via this interface
            gateway: ""             # Optional explicit gateway; auto-discovered if empty
            useDefaultRoute: false  # Copy default route from main table instead
```
