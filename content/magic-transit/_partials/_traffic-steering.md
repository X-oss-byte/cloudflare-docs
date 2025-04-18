---
_build:
  publishResources: false
  render: never
  list: never
inputParameters: productName;;greURL;;staticRoutesURL;;ipsecURL
---

# Traffic steering

$1 uses a static configuration to route traffic through Anycast tunnels using the [Generic Routing Encapsulation (GRE)]($2) and [Internet Protocol Security (IPsec)]($4) protocols from Cloudflare’s global network to your network, and from your network to Cloudflare’s global network.

$1 steers traffic along tunnel routes based on priorities you define in the Cloudflare dashboard or via API.

The example in this diagram has three tunnel routes. Tunnels 1 and 2 have the top priority routes, and Tunnel 3 has a secondary priority route.

```mermaid
flowchart LR
accTitle: Tunnels diagram
accDescr: The example in this diagram has three tunnel routes. Tunnels 1 and 2 have top priority and Tunnel 3 is secondary.

subgraph Cloudflare
direction LR
B[Cloudflare <br> data center]
C[Cloudflare <br> data center]
D[Cloudflare <br> data center]
end

A((User)) --> Cloudflare --- E[Anycast IP]
E[Anycast IP] --> F[/Tunnel 1 / <br> priority 1/] --> I{{Customer <br> data center/ <br> network 1}}
E[Anycast IP] --> G[/Tunnel 2 / <br> priority 1/] --> J{{Customer <br> data center/ <br> network 2}}
E[Anycast IP] --> H[/Tunnel 3 / <br> priority 2/] --> K{{Customer <br> data center/ <br> network 3}}
```
<br />

When there are multiple routes to the same prefix with equal priority and different next-hops, Cloudflare uses equal-cost multi-path (ECMP) routing. An example of multiple routes with equal priority would be Tunnel 1 and Tunnel 2.

The use of ECMP routing provides load balancing across tunnels with routes of the same priority.

## Equal-cost multi-path routing

Equal-cost multi-path routing uses hashes calculated from packet data to determine the route chosen. The hash always uses the source and destination IP addresses. For TCP and UDP packets, the hash includes the source and destination ports as well. The ECMP algorithm divides the hash for each packet by the number of equal-cost next hops. The modulus (remainder) determines the route the packet takes.

Using ECMP has a number of consequences:
- Routing to equal-cost paths is probabilistic.
- Packets in the same session (or flow) with the same source and destination have the same hash. The packets also use the same next hop.
- Routing changes in the number of equal-cost next hops can cause traffic to use different tunnels. For example, dynamic reprioritization triggered by health check events can cause traffic to use different tunnels.

As a result, ECMP provides load balancing across tunnels with the same prefix and priority.

{{<Aside type="note" header="Note:">}}
Packets in the same flow use the same tunnel unless the tunnel priority changes. Packets for different flows can use different tunnels depending on which tunnel the flow’s 4-tuple – source and destination IP and source and destination port – hash to.
{{</Aside>}}

### Examples

This diagram illustrates how ECMP distributes traffic equally across two paths with the same prefix and priority.

#### Normal traffic flow

```mermaid
flowchart LR
accTitle: Tunnels diagram
accDescr: This example has three tunnel routes, with traffic equally distributed across two paths.

subgraph Cloudflare
direction LR
B[Cloudflare <br> data center]
C[Cloudflare <br> data center]
D[Cloudflare <br> data center]
end

Z("Load balancing for some <br> priority tunnels uses ECMP <br> (hashing on src IP, dst IP, <br> scr port, dst port)") --- Cloudflare
A((User)) --> Cloudflare --- E[Anycast IP]
E[Anycast IP] --> F[/"GRE Tunnel 1 / <br> priority 1 / <br> ~50% of flows"/] --> I{{Customer <br> data center/ <br> network 1}}
E[Anycast IP] --> G[/"GRE Tunnel 2 / <br> priority 1 / <br> ~50% of flows"/] --> J{{Customer <br> data center/ <br> network 2}}
E[Anycast IP] --> H[/GRE Tunnel 3 / <br> priority 2 / <br> 0% of flows/] --o K{{Customer <br> data center/ <br> network 3}}
```
<br />

When $1 health checks determine that Tunnel 2 is unhealthy, that route is dynamically de-prioritized, leaving Tunnel 1 with the sole top-priority route. As a result, traffic is steered away from Tunnel 2, and all traffic flows to Tunnel 1.

####  Failover traffic flow: Scenario 1

Customer router failure

```mermaid
flowchart LR
accTitle: Tunnels diagram
accDescr: This example has Tunnel 2 unhealthy, and all traffic prioritized to Tunnel 1.

subgraph Cloudflare
direction LR
B[Cloudflare <br> data center]
C[Cloudflare <br> data center]
D[Cloudflare <br> data center]
end

Z(Tunnel health is <br> determined by <br> health checks that <br> run from all Cloudflare <br> data centers) --- Cloudflare
A((User)) --> Cloudflare --- E[Anycast IP]
E[Anycast IP] --> F[/"Tunnel 1 / <br> priority 1 / <br> ~100% of flows"/]:::green --> I{{Customer <br> data center/ <br> network 1}}
E[Anycast IP] --> G[/Tunnel 2 / <br> priority 3 / <br> unhealthy / 0% of flows/]:::red --x J{{Customer <br> data center/ <br> network 2}}
E[Anycast IP] --> H[/Tunnel 3 / <br> priority 2 / <br> 0% of flows/] --o K{{Customer <br> data center/ <br> network 3}}
classDef red fill:#EE4B2B
classDef green fill:#00FF00
```
<br />

When $1 determines that Tunnel 1 is unhealthy as well, that route is also de-prioritized, leaving Tunnel 3 with the top priority route. In that case, all traffic flows to Tunnel 3.

####  Failover traffic flow: Scenario 2

Intermediary ISP failure

```mermaid
flowchart LR
accTitle: Tunnels diagram
accDescr: This example has Tunnel 1 and 2 unhealthy, and all traffic prioritized to Tunnel 3.

subgraph Cloudflare
direction LR
B[Cloudflare <br> data center]
C[Cloudflare <br> data center]
D[Cloudflare <br> data center]
end

Z(Lower-priority tunnels <br> are used when <br> higher-priority tunnels <br> are unhealthy) --- Cloudflare
A((User)) --> Cloudflare --- E[Anycast IP]
E[Anycast IP]  -- Intermediary <br> network issue -->  F[/Tunnel 1 / <br> priority 3 / <br> unhealthy / 0% of flows/]:::red --x I{{Customer <br> data center/ <br> network 1}}
E[Anycast IP]  -- Intermediary <br> network issue -->  G[/Tunnel 2 / <br> priority 3 / <br> unhealthy / 0% of flows/]:::red --x J{{Customer <br> data center/ <br> network 2}}
E[Anycast IP] -->  H[/Tunnel 3 / <br> priority 2 / <br> 100% of flows/]:::green --> K{{Customer <br> data center/ <br> network 3}}
classDef red fill:#EE4B2B
classDef green fill:#00FF00
```
<br />

When $1 determines that Tunnels 1 and 2 are healthy again, it re-prioritizes those routes, and traffic flow returns to normal.

## ECMP and bandwidth utilization

Because ECMP is probabilistic, the algorithm routes roughly the same number of flows through each tunnel. However it does not consider the amount of traffic already sent through a tunnel when deciding where to route the next packet.

For example, consider a scenario with many very low-bandwidth TCP connections and one very high-bandwidth TCP connection. Packets for the high-bandwidth connection have the same hash and thus use the same tunnel. As a result, that tunnel utilizes greater bandwidth than the others.

{{<Aside type="note" header="Note">}}

$1 supports a weight field that you can apply to a route so that a specified percentage of traffic uses a certain tunnel rather than other equal-cost tunnels. Refer to [Configure static routes]($3) for more information.

For example, in a scenario where you want to route 70% of your traffic through ISP A and 30% through ISP B, you can use the weight field to help achieve that.

Note that because ECMP balances flows probabilistically, the use of weights is only approximate.

For more on $1 tunnel weights, contact your Cloudflare customer service manager.

{{</Aside>}}
