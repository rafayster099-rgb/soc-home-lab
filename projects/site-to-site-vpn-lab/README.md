# Site-to-Site IPsec VPN Lab (Proxmox + pfSense)

A hands-on lab simulating two office locations connected over an encrypted Site-to-Site VPN tunnel, built entirely from scratch in Proxmox using pfSense as the routing/firewall platform at each site.

## Objective

Build two isolated, independent networks representing two company sites, connect them over a simulated WAN link, and establish an IPsec VPN tunnel so that a device on Site A's LAN can securely reach a device on Site B's LAN — with packet-level proof that traffic crossing the shared link is genuinely encrypted, not just labeled as a VPN.

## Topology

```
        ISP / Home Network (192.168.2.0/24)
                    |
              [ pfSense A ]
         WAN: 192.168.2.51/24
         LAN: 10.0.0.1/24  ── Site A LAN (10.0.0.0/24)
                                  └─ Windows Server: 10.0.0.101
        OPT1: 172.16.99.1/30
                    |
            Transit Switch (vmbr3)
              172.16.99.0/30
                    |
        WAN: 172.16.99.2/30
              [ pfSense B ]
         LAN: 10.1.0.1/24  ── Site B LAN (10.1.0.0/24)
                                  ├─ Ubuntu: 10.1.0.100
                                  └─ Kali: 10.1.0.101/102
```

**Environment:** Proxmox VE, pfSense 2.8.1-RELEASE (x2), Ubuntu Server, Kali Linux

**Isolated bridges built for this project:**
- `vmbr2` — Site B LAN
- `vmbr3` — Transit link between the two pfSense routers (no DHCP, no LAN devices — pure point-to-point)

## What Was Built

1. Two independent, fully isolated LANs (10.0.0.0/24 and 10.1.0.0/24), each with its own pfSense router and working DHCP
2. A dedicated /30 transit link between the two routers' OPT1/WAN interfaces, simulating the public internet segment between two real offices
3. Firewall rules permitting traffic across the transit interfaces and the tunnel itself
4. IPsec Site-to-Site VPN (IKEv2, Mutual PSK, AES-256/SHA-256, Phase 2 with AES256-GCM) connecting the two sites
5. Verified end-to-end LAN-to-LAN connectivity through the tunnel
6. Packet capture proof that traffic crossing the shared transit link is fully encrypted (ESP), not plaintext

## IPsec Configuration Summary

| Setting | pfSense A | pfSense B |
|---|---|---|
| Interface | OPT1 | WAN |
| Local IP | 172.16.99.1 | 172.16.99.2 |
| Remote Gateway | 172.16.99.2 | 172.16.99.1 |
| Phase 1 | IKEv2, Mutual PSK, AES-256, SHA-256, DH Group 14 | (identical) |
| Phase 2 | Local: 10.0.0.0/24, Remote: 10.1.0.0/24, AES256-GCM | Local: 10.1.0.0/24, Remote: 10.0.0.0/24, AES256-GCM |

## Verification

- **Status → IPsec**: Phase 1 shows `Established`; Phase 2 shows `Installed` with active byte counters on both sides
- **Cross-site ping**: `10.0.0.101 → 10.1.0.102` succeeds (0% packet loss)
- **Packet capture on the transit interface (OPT1)** during the ping test shows only:
  ```
  IP 172.16.99.1 > 172.16.99.2: ESP(spi=0xc86f5f5b, seq=0x22), length 96
  IP 172.16.99.2 > 172.16.99.1: ESP(spi=0xce517fe0, seq=0x22), length 96
  ```
  No ICMP, no visible source/destination host addresses, no indication of what protocol or hosts are actually communicating — confirming the tunnel is genuinely encrypting traffic end-to-end, not just routing it.

## Problems Encountered & Root-Caused

Documenting these because the troubleshooting was the actual learning — not just following steps.

1. **WAN/LAN interface assignment got flipped on first boot of the cloned router.** Diagnosed and corrected by cross-referencing MAC addresses between Proxmox's Hardware tab and the pfSense console's interface list, rather than assuming interface order.

2. **DHCP server silently stopped running after an interface reassignment**, on both routers independently, at different points in the project. No error was shown — the symptom was a client stuck on an APIPA address. Confirmed via `ps aux | grep dhcpd` showing no live process, then fixed by re-running the interface IP/DHCP setup and explicitly re-enabling the DHCP service.

3. **Transit link ping failed despite correct IPs on both routers.** Diagnosed layer by layer instead of guessing:
   - **ARP** confirmed Layer 2 connectivity was fine (both routers saw each other's MAC address)
   - **Traceroute** showed zero response at any hop, ruling out a routing issue
   - **Firewall logs** revealed the actual cause: pfSense's default "Block private networks and loopback addresses" option on a WAN-type interface was silently dropping all traffic from the private transit subnet

4. **IPsec tunnel established (Phase 1 + Phase 2 both "up"), but cross-site ping still failed.** Isolated the cause using IPsec's own Phase 2 byte counters: outbound bytes were incrementing but inbound stayed at zero, proving the packet was reaching the remote router and being decrypted, but never reaching the LAN. Root cause: the **Firewall → Rules → IPsec** rule had its Source/Destination reversed — a separate, easy-to-miss rule set from the router's normal interface rules, since IPsec-decrypted traffic is filtered on its own dedicated tab.

## Skills Demonstrated

- Network segmentation and isolated multi-site design in a virtualized environment
- Router/firewall interface configuration and troubleshooting (pfSense)
- IPsec VPN configuration (Phase 1/Phase 2, IKEv2, PSK authentication)
- Systematic, layer-based troubleshooting methodology (Layer 2 → Layer 3 → firewall/application layer)
- Packet analysis and verification using tcpdump/pfSense packet capture
- DHCP, NAT, and firewall rule administration

## Next Steps

- OSPF, ACL, and NAT labs (in progress)
- Expand into a multi-site mega-lab combining routing, segmentation, VPN, and Active Directory into a single integrated environment
