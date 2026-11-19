# Kamui Routing

Kamui Routing provides two Bash controllers for sending selected downstream
IPv4 traffic through a Linux gateway to a remote WireGuard Internet exit. This
repository is a sanitized public derivative of privately developed and
operated Linux/WireGuard infrastructure. The addresses, networks, interface
names, and node labels below describe an example. Supply your own topology and
WireGuard configuration.

## Origins and Development

Kamui Routing emerged from an evolving multi-site networking
environment initially focused on remote connectivity, VPN
routing, and network-wide DNS filtering. The infrastructure
gradually expanded to incorporate isolated test networks,
remote gateways, and portable network devices, introducing
increasingly complex routing requirements.

Two distinct use cases ultimately shaped the project:
selectively routing traffic from an Ethernet-connected
downstream network through a remote exit, and forwarding
traffic from a WireGuard-connected router back through the
same tunnel interface toward an alternate exit. These
requirements led to independent policy-routing controllers,
subsequently extracted and generalized from the private
infrastructure for public use.

## Architecture

| Controller | Example path through the gateway | Source matched at the gateway | Policy table / priority |
| --- | --- | --- | --- |
| `gedo-obito` | Ethernet router (Gedo Mazo) → `lan0` → `wg-exit` → exit (Obito) | `10.240.0.2/32` | 145 / 100 |
| `harashin-obito` | Travel router (Flying Raijin) → `wg-exit` → same `wg-exit` interface → exit (Obito) | `10.240.145.5/32` | 146 / 110 |

The gateway (Nagato) uses source-based policy routing to select an IPv4 default
route through WireGuard. Both controllers masquerade their selected source on
WireGuard egress. The remote exit must forward that traffic and provide
Internet-side NAT or equivalent return routing. If a downstream router sends
packets with its clients' original source addresses, those packets will miss
these single-address rules. The example assumes downstream source NAT when
clients use other addresses.

## Prerequisites

- A Linux gateway with Bash, iproute2, WireGuard tools, nftables, and standard
  utilities including `awk`. Gedo also uses `/usr/sbin/sysctl`. Harashin uses
  `iptables-nft` with the comment match.
- Administrator privileges and IPv4 forwarding on the gateway. Gedo enables
  the global forwarding sysctl and leaves it enabled on deactivation. Enable it
  separately before activating Harashin.
- A working WireGuard configuration on the gateway, exit, and travel router
  as applicable. The exit peer on the gateway must own the public IPv4
  destinations that should go through it, typically via `0.0.0.0/0`. The travel
  peer needs the exact configured source `/32` in its AllowedIPs and must send
  the desired destinations through the tunnel. WireGuard selects the outbound
  peer by packet destination. An interface-only route does not select a peer
  by its tunnel IP.
- Manage WireGuard's automatically installed routes and rules (for example,
  with `Table = off` in a manually routed `wg-quick` setup). Provide the
  gateway's connected routes, return routes to the downstream sources, and an
  underlay route to the exit endpoint. Check that other policy rules and the
  gateway's ordinary default route behave as intended.
- A compatible forwarding firewall. Gedo adds nftables accepts for its
  outbound flow and established or related replies. Harashin inserts a
  source-specific WireGuard-to-WireGuard ACCEPT at the head of the
  `iptables-nft` FORWARD chain. The host firewall still needs to permit replies
  and any intended local forwarding. An accept in one nftables base chain
  leaves other base chains free to drop the packet. Review existing NAT rules
  as well, since a matching NAT rule can establish a mapping first.

## Configure and install

```sh
git clone https://github.com/wdharri2/kamui-routing.git
cd kamui-routing
```

Edit the constants at the top of each controller before installing. The example
gateway WireGuard address is `10.240.145.1` and the exit tunnel address is
`10.240.145.4`. Gedo uses `lan0` for downstream Ethernet. Harashin routes
`10.240.145.0/29` through WireGuard, `10.240.0.0/30` through `lan0`, and
`172.20.100.0/24` through the ordinary uplink `wan0`. Its firewall guard exempts those two prefixes and
`10.240.10.0/24` by destination only, regardless of
ingress or egress interface. Table 146 has no specific
route for `10.240.10.0/24`, so that exception does not
establish reachability. Gedo has no controller-defined
local-network exceptions.

Reserve both routing tables, policy priorities, and the nftables table names
`gedo_obito` and `harashin_obito` for these controllers. Activation replaces
the named nftables table and overwrites the example routes in the configured
policy table. Deactivation flushes the entire configured routing table.
Deactivate using the old settings before changing addresses, interfaces,
priorities, or table IDs.

```sh
sudo install -m 755 scripts/gedo-obito /usr/local/sbin/gedo-obito
sudo install -m 755 scripts/harashin-obito /usr/local/sbin/harashin-obito
```

## Activate, inspect, and deactivate

```sh
sudo gedo-obito on
sudo gedo-obito status
sudo gedo-obito off

sudo harashin-obito on
sudo harashin-obito status
sudo harashin-obito off
```

Run one operation at a time. Gedo installs policy
routing before its firewall and removes the firewall
before its policy route; temporary fallback is possible.
Harashin installs its firewall before routing and removes
routing first. Its untagged `iptables-nft` FORWARD rule
is removed by exact match, potentially removing an
identical preexisting rule.

Use `status` to inspect the policy rule, routing table, simulated IPv4 route,
and firewall state. Check WireGuard peer handshakes and counters, then test
IPv4 egress, DNS, IPv6, and the intended failure behavior from behind each
downstream router on a test network you control. If traffic stalls, check
AllowedIPs, endpoint routing, forwarding in both directions, NAT on both hops,
and tunnel MTU. A simulated route lookup does not verify packet delivery.

## Failure behavior and scope

- The guard only filters forwarded IPv4 packets from the configured ingress
  and source. It drops matching packets routed outside WireGuard, subject to
  Harashin's local exceptions. It does not cover gateway-originated traffic,
  other sources or ingress interfaces, or paths outside this gateway.
- Activation checks the route to the exit's tunnel address. Harashin also
  searches AllowedIPs for the travel prefix without verifying its peer identity. Neither controller
  verifies exit liveness, checks which peer owns arbitrary public
  destinations, or monitors tunnel health. If WireGuard remains the selected
  interface and the exit stops passing traffic, packets can stall there. The
  guard remains relevant if Linux falls back to a different output interface.
- IPv6 and DNS have no dedicated controls. DNS sent as matching forwarded IPv4
  traffic follows the same policy as other IPv4 traffic. Queries handled by
  the gateway or sent over an alternate router path can use other egress.
- nftables rules are changed through separate commands, not one atomic replacement. Firewall, policy routing, and
  forwarding changes span separate operations. Failed activation can leave
  a guard, routes, forwarding rules, or the forwarding sysctl in place.
  Inspect state and use `off` with the same settings to clean up. Route
  cache and table flush errors are ignored, so verify the table after cleanup.
  No automatic rollback, reboot persistence, or concurrent-operation
  protection is provided.

## Development

Run `bash scripts/check` for offline syntax validation and ShellCheck when
available. The check does not run either controller or change network state.

Licensed under the [MIT License](LICENSE).
