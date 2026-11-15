# EIGRP / OSPF / BGP Troubleshooting Cheat Sheet

## 1. Protocol Quick-Facts Table (memorize these first)

| | EIGRP | OSPF | BGP |
|---|---|---|---|
| **AD (default)** | 90 (internal) / 170 (external) | 110 | 20 (eBGP) / 200 (iBGP) |
| **Transport** | IP proto 88 (own transport) | IP proto 89 (own transport) | TCP/179 |
| **Metric** | Composite (BW, delay, [rel/load]) | Cost = ref-BW / interface BW | Path attributes (no true "metric") |
| **Hello/Dead default** | 5 sec / 15 sec (60/180 on NBMA/slow) | 10 sec / 40 sec (broadcast/P2P); 30/120 (NBMA) | Hello 60 sec / Hold 180 sec |
| **Adjacency requires match** | AS #, K-values, primary subnet | Area #, Hello/Dead, area type (stub flags), MTU | AS # (per-neighbor), reachable peer IP |
| **Convergence tool** | DUAL (feasible successor) | SPF (Dijkstra) | Best-path algorithm (not shortest-path) |

---

## 2. EIGRP Troubleshooting

**Neighbor won't come up — check in this order:**
- [ ] `show ip eigrp neighbors` — is it there at all?
- [ ] **AS number mismatch** — most common cause, no log message, just silence
- [ ] **K-value mismatch** → causes "K-value mismatch" log; use `show ip protocols` to compare
- [ ] Interface passive (`passive-interface`) — kills hellos silently
- [ ] Primary IP not on same subnet (secondary IPs don't form adjacency)
- [ ] ACL blocking proto 88 or multicast 224.0.0.10

**Routes missing from table:**
- Check **feasibility condition**: reported distance (RD) from neighbor must be < your own FD to that network, or it's not a feasible successor → goes to "stuck in active" risk
- `show ip eigrp topology` — look for routes stuck as "Query" (SIA candidates)
- Auto-summary (legacy, off by default in modern IOS but check anyway) can black-hole discontiguous networks
- Split-horizon on hub-spoke (Frame Relay/DMVPN) — routes learned in won't be re-advertised out same interface

⚠️ **Gotcha:** EIGRP will NOT form neighbors if K-values differ, but WILL happily show a neighbor as "up" with wrong AS silently ignored (no adjacency at all — easy to mistake for a down interface).

---

## 3. OSPF Troubleshooting

**Neighbor stuck in a state — the state tells you the problem:**

| Stuck State | Likely Cause |
|---|---|
| **Down/Attempt** | No hellos received — L1/L2, ACL, wrong network stmt |
| **Init** | Hearing hellos, not seeing itself in neighbor's hello → one-way comms, ACL asymmetry |
| **2-Way** | Normal for DROther-DROther on broadcast; stuck here on P2P = fine actually, but on broadcast expected |
| **ExStart/Exchange** | **MTU mismatch** (classic exam trap) — DBD packets sent but stuck negotiating master/slave |
| **Loading** | LSR/LSU issue, usually transient; persistent = packet loss |
| **Full** | Working! |

**Other adjacency blockers:**
- [ ] Area number mismatch on the link
- [ ] **Area type mismatch** — one side stub, other side normal (flags in hello must match: E-bit, N-bit)
- [ ] Hello/Dead timer mismatch (must match exactly, unlike some other params)
- [ ] Network type mismatch (broadcast vs point-to-point vs NBMA) → wrong DR/BDR behavior or hellos to wrong address
- [ ] Duplicate Router-ID on the network
- [ ] Passive-interface set

**Route issues:**
- `show ip ospf database` before `show ip route` — confirm LSA exists before assuming routing bug
- Stub/Totally Stubby/NSSA — check what's allowed to leak (NSSA allows redistribution + Type-7→5 translation at ABR)
- Virtual-links needed if area doesn't touch Area 0 directly

⚠️ **Gotcha:** MTU mismatch lets neighbors reach 2-Way / Init fine — it only breaks at **ExStart/Exchange**. If a student sees "stuck at Exchange," MTU is the #1 suspect.

⚠️ **Gotcha:** Stub area flags mismatch won't even get you past Down — it's silently rejected during hello processing, easy to confuse with a simple area-number typo.

---

## 4. BGP Troubleshooting

**Neighbor states (like OSPF, the state = the clue):**

| State | Meaning / Likely Issue |
|---|---|
| **Idle** | Not even trying — check config exists, or manual reset |
| **Connect/Active** | TCP handshake failing — reachability, ACL, wrong neighbor IP, or missing static route to non-directly-connected eBGP peer |
| **OpenSent/OpenConfirm** | TCP up, but **AS number mismatch** or hold-timer mismatch |
| **Established** | Working! |

**Common blockers:**
- [ ] **eBGP multihop** missing when peering to a loopback / non-directly-connected neighbor
- [ ] `update-source` not set for iBGP peering via loopback
- [ ] Neighbor statement AS doesn't match remote's actual AS
- [ ] ACL/firewall blocking TCP/179
- [ ] No route to neighbor IP in routing table (BGP needs an IGP/static route first — BGP doesn't create its own reachability)

**Routes not advertised/received — the classic BGP trap list:**
- **Synchronization** (legacy, usually off, but check if old IOS)
- **Next-hop unreachable** — eBGP-learned next-hop not reachable via IGP (very common in labs), especially on multi-access segments where next-hop isn't reset
- **iBGP split-horizon rule**: routes learned from an iBGP peer are **NEVER** re-advertised to another iBGP peer (must use route-reflector or full mesh) — huge exam trap
- `neighbor x.x.x.x next-hop-self` missing on eBGP→iBGP redistribution point
- Prefix filtered by `distribute-list`/`prefix-list`/`route-map` — check both **inbound AND outbound** sides
- Route present in BGP table (`show ip bgp`) but not in RIB (`show ip route`) → another route with better AD, or best-path not selected yet

**Best-path selection (order — memorize top 4 minimum):**
1. Highest **Weight** (Cisco-local only)
2. Highest **Local Preference**
3. Locally originated
4. Shortest **AS-Path**
5. Lowest Origin code (IGP < EGP < Incomplete)
6. Lowest MED
7. eBGP over iBGP
8. Lowest IGP metric to next-hop
9. Oldest route (if eBGP)
10. Lowest Router ID

⚠️ **Gotcha:** BGP will happily sit in **Active** state forever with zero useful log output if it simply can't reach the peer IP — students often assume "Active = working," but Active is actually a *failure* state (it means "actively retrying TCP").

⚠️ **Gotcha:** A route can be 100% correctly received and best-path-selected in BGP but still missing from the RIB — always check `show ip route` separately, don't assume BGP table = routing table.

---

## 5. Cross-Protocol "Easy to Confuse" Section

| Confusable Pair | The Difference |
|---|---|
| EIGRP AD 90 vs OSPF AD 110 | Lower AD wins if both protocols learn same route — EIGRP internal wins over OSPF by default |
| OSPF Cost vs EIGRP Metric | OSPF = simple bandwidth-based formula; EIGRP = composite formula (BW+delay by default) — don't try to compare them directly |
| OSPF "stuck at 2-Way" vs "stuck at Exchange" | 2-Way on broadcast segments between DROthers is **normal**, not a fault. Exchange getting stuck = MTU. |
| EIGRP Stuck-in-Active vs OSPF slow convergence | SIA = EIGRP query never answered (usually a flaky neighbor down the chain) — unique to EIGRP, no OSPF equivalent |
| iBGP vs eBGP AD | iBGP = 200 (very high, easily beaten by IGPs) — a route can be "correct" in BGP but never installed because an IGP route to same prefix has lower AD |
| iBGP split-horizon vs EIGRP split-horizon | Totally different mechanisms — EIGRP's is interface-based (don't advertise back out same interface); BGP's is peer-type-based (don't re-advertise iBGP-learned routes to other iBGP peers) |
| Passive-interface effect | EIGRP/OSPF: suppresses hellos = no adjacency at all on that interface. BGP: no such thing as passive-interface — BGP neighbors are explicitly configured peer-to-peer regardless of interface |

---

## 6. Fastest Diagnostic Commands (in order of usefulness)

**EIGRP:** `show ip eigrp neighbors` → `show ip eigrp topology` → `show ip protocols` → `debug eigrp packets` (if desperate)

**OSPF:** `show ip ospf neighbor` → `show ip ospf interface` (check network type/timers/area) → `show ip ospf database` → `debug ip ospf adj`

**BGP:** `show ip bgp summary` (state column!) → `show ip bgp neighbors x.x.x.x` (look for "last reset reason") → `show ip bgp` (best-path check) → `show ip route bgp`
