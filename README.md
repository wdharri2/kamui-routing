# WireGuard Exit Router

Linux routing controllers for directing downstream router
traffic through a remote WireGuard Internet exit.

The scripts demonstrate source-based routing, firewall
forwarding, NAT masquerading, and exit-routing kill switches.

Node names are retained from the original implementation.
All addresses and interface assignments in this repository
are illustrative, not live network configuration.

## Architecture

    Gedo Mazo (Ethernet)
              |
             lan0
              |
            Nagato
              |
           wg-exit
              |
            Obito
              |
           Internet

    Flying Raijin (WireGuard)
              |
           wg-exit
              |
            Nagato
              |
           wg-exit
              |
            Obito
              |
           Internet

Nagato is the Linux routing gateway.

Obito is a remote WireGuard peer providing Internet egress.

Gedo Mazo is an Ethernet-connected downstream router.

Flying Raijin is a WireGuard-connected travel router.

## Requirements

- Linux gateway with administrator privileges
- WireGuard configured and connected
- iproute2
- nftables
- iptables-nft
- IPv4 forwarding
- Remote exit configured for forwarding and NAT

These scripts assume an existing compatible Linux routing
and firewall environment.

## Example configuration

Illustrative addresses:

    Nagato WireGuard:  10.240.145.1
    Obito WireGuard:   10.240.145.4
    Flying Raijin:     10.240.145.5
    Gedo Mazo:         10.240.0.2

Illustrative interfaces:

    lan0       Downstream Ethernet
    wan0       Ordinary Internet uplink
    wg-exit    WireGuard network

These addresses and interfaces must be adjusted for the
target system before installation.

The scripts currently contain configuration constants
and network-specific routing assumptions.

## Installation

Clone the repository:

    git clone git@github.com:wdharri2/wireguard-exit-router-examples.git
    cd wireguard-exit-router-examples

Edit both scripts to match the intended network.

Verify the required WireGuard peers, addresses, interfaces,
firewall configuration, and remote forwarding capabilities.

Install the controllers:

    sudo install -m 755 scripts/gedo-obito /usr/local/sbin/gedo-obito
    sudo install -m 755 scripts/harashin-obito /usr/local/sbin/harashin-obito

## Usage

Gedo Mazo:

    sudo gedo-obito on
    sudo gedo-obito off
    sudo gedo-obito status

Flying Raijin:

    sudo harashin-obito on
    sudo harashin-obito off
    sudo harashin-obito status

## How it works

### Policy routing

Each downstream router receives an independent
source-based routing policy.

Gedo Mazo uses table 145.

Flying Raijin uses table 146.

Selected IPv4 traffic is directed through the WireGuard
interface rather than the gateway's ordinary default route.

### WireGuard

WireGuard encrypts traffic between participating peers.

The remote exit peer must advertise the necessary
destination ranges through AllowedIPs.

Traffic from another WireGuard peer may enter and exit
through the same Linux WireGuard interface.

### Firewall forwarding

The gateway permits selected forwarded traffic while
restricting unintended Internet fallback.

An existing default-drop FORWARD chain may require
additional explicit forwarding rules.

An ACCEPT verdict in one nftables base chain does not
override a DROP verdict in another base chain.

### NAT masquerading

Source NAT translates forwarded downstream traffic onto
the gateway's WireGuard address.

The remote exit must then forward or translate packets
through its own Internet connection.

### Kill switches

The controllers install IPv4 firewall rules intended to
prevent selected traffic from escaping through an
unintended gateway interface.

These protections do not constitute a complete IPv6 or
DNS leak-prevention implementation.

## Verification

Test from a device behind the downstream router:

    curl -4 https://api.ipify.org

Verify the Linux routing rules:

    ip rule show

Inspect routing tables:

    ip route show table 145
    ip route show table 146

Inspect WireGuard:

    sudo wg show

Inspect nftables:

    sudo nft list ruleset

## Troubleshooting

If connectivity fails, inspect:

- WireGuard handshakes
- Source-based routing rules
- Firewall forwarding rules and counters
- NAT configuration on both gateway and exit
- DNS connectivity
- IPv6 behavior
- Tunnel MTU/MSS

## Limitations

These are illustrative routing controllers, not a
general-purpose installer.

- Network values require customization
- Remote exit provisioning is not included
- IPv6 routing is not implemented
- Reboot persistence is not installed
- No automatic rollback is provided
- Existing firewall compatibility requires verification

Review and test all changes before deploying them to
an unfamiliar network.
