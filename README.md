# bess-cndp-commands

## AF_XDP — the high-speed socket bessd uses
AF_XDP (Address Family eXpress Data Path) is a Linux socket family that bypasses the kernel network stack. The socket (called an xsk) attaches to one specific RX queue of one specific netdev. A small XDP/BPF program runs at the very start of the RX path and decides: XDP_REDIRECT (push the packet into the xsk's userspace ring) or XDP_PASS (let the kernel continue normal processing).

bessd's CNDP layer attaches one xsk per port to queue 0 with the xsk_def_xdp_prog BPF program. So packets that arrive on queue 0 are redirected into bessd; packets on any other queue (1..47) hit the default XDP_PASS and go to the kernel stack — but the pod netns has no user-space consumer for them, so they're silently dropped.
```

  Wire ──► access PF ──► RSS (hash → queue) ──► one of 48 queues
                                                      │
                                  ┌───────────────────┼───────────────────┐
                                  ▼                    ▼                   ▼
                              queue 0              queue 1..47          queue 47
                                  │                    │                   │
                            xsk_def_xdp_prog       (no BPF on these — go to kernel,
                                  │                 kernel has nothing to do with them
                                  ▼                 in the pod netns → DROPPED)
                            bessd (CNDP)


```
### Why our packets were invisible
Our GTP-U has a single, narrow 5-tuple:


  src 192.168.252.5  : 2152  →  dst 192.168.252.3 : 2152   proto UDP
The Toeplitz hash is deterministic — it's literally a function f(5-tuple) = constant. Every OCUDU GTP-U packet hashes to the same queue. That single number could be anything from 0 to 47. There was no statistical reason for it to land on q0. In our case it didn't — every GTP-U packet went to some other queue, the kernel dropped it, bessd never saw it. Result: cndp_port0 RX was growing (from LLDP multicast, which has special MAC-based delivery), but pktParse → pdrLookup was zero.

The classic mistake when looking at this: cndp_port0 Inc/RX is increasing, so you assume "packets are arriving at bess". They are — some are — but they're the LLDP/broadcast noise. The specific GTP-U flow you care about is on another queue and the kernel is throwing it away.

### The two ethtool fixes — what each one really does
* Fix A — n-tuple flow steering (ethtool -N)

```
ethtool -K access ntuple on                                  # enable n-tuple table
```
```
ethtool -N access flow-type udp4 dst-port 2152 action 0      # explicit rule
```
n-tuple is a priority-1 table in the NIC, evaluated before RSS. Any UDP packet with dst port 2152 — regardless of its hash — gets steered directly to queue 0. Surgical: only catches GTP-U; everything else still uses normal RSS.

We used this for the uplink (access PF) because the uplink is one well-known UDP port. Worked instantly: pdrLookup gate 1 (UL match) went from 0 → 322.

* Fix B — collapse the RSS indirection table (ethtool -X equal 1)
```
ethtool -X access equal 1
```
```
ethtool -X core   equal 1
```
This rewrites the 64-entry RSS indirection table so that every bucket points at queue 0. No matter what the hash computes to, the table sends it to q0. Functionally identical to "no RSS, single queue".

We used this on the core PF for downlink because n-tuple wasn't supported on that PF's driver, and downlink return traffic (any UE-bound IP from the internet) doesn't have a fixed UDP port — much harder to steer with n-tuple.

Why we couldn't just do ethtool -L combined 1
That's the textbook way ("just give me one queue"). It rebuilds the queue layout. But the operation tears down all queues and rebuilds them — which means the kernel needs to detach any active xsk sockets. bessd's xsk is bound to queue 0 and bessd doesn't release it. → Device or resource busy.

You can do -L combined 1 if you first stop bessd, but that breaks the PFCP session and forces a full datapath rebuild. -X equal 1 achieves the same effective behaviour at the indirection layer without touching the queues — bessd's xsk stays bound to q0, and all packets now land in q0. That's why this is the AF_XDP-friendly fix.

### Recap
Hardware: 48 RX queues per PF.
RSS hashes our specific GTP-U flow to one queue 0-47. Not q0 in our case.
AF_XDP/CNDP attaches bessd to q0 only.
Result: every OCUDU GTP-U packet lands on a queue bessd doesn't watch → kernel can't do anything with it in the pod netns → dropped.
From outside, the symptom looks like "PDR doesn't match" because pdrLookupFail cumulative is non-zero (from packets that did hit q0 in earlier moments when the topology was slightly different) — but in reality almost nothing was reaching pdrLookup at all.
n-tuple + -X equal 1 force every packet of interest to q0 → bessd finally sees them.

### resources 

[](https://www.kernel.org/doc/html/latest/networking/scaling.html)

[](https://www.kernel.org/doc/html/latest/networking/af_xdp.html)

[](https://github.com/xdp-project/xdp-tutorial)

