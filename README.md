# Networking & Cloud Labs

Hands-on labs for learning Linux, networking, and cloud fundamentals, built toward becoming a stronger cloud engineer. Each lab builds on the previous one, and every lab includes the problems I hit and how I fixed them, since the debugging is where most of the learning happened.

| Lab | Topic | Stack |
|---|---|---|
| [Lab 1](#lab-1--linux-router-with-nat) | Linux router (NAT + forwarding) between a home network and an isolated lab network | VirtualBox, Ubuntu Server, netplan, iptables |
| [Lab 2](#lab-2--hybrid-cloud-site-to-site-ipsec-vpn) | Site-to-site IPsec VPN between the home lab and an AWS VPC | AWS (VPC, EC2, SSM), StrongSwan, Libreswan |

---

## Lab 1 — Linux Router with NAT

**Goal:** Build a small virtual lab where one Ubuntu Server VM acts as a router between the home network and an isolated internal network, with two client VMs behind it.

### Topology

```
[Home network] --- 192.168.x.x --- [router-vm] --- 10.10.10.1 --- [Host-only network]
                                    (enp0s3)       (enp0s8)             |
                                                              [client1: 10.10.10.10]
                                                              [client2: 10.10.10.11]

Windows host also joins the internal network at 10.10.10.254
```

### What I built

- `router-vm` with two adapters: Bridged (internet-facing) and Host-only (lab-facing, static `10.10.10.1/24`)
- IP forwarding (`net.ipv4.ip_forward`) plus `iptables` MASQUERADE so lab clients can reach the internet
- Two clients with static IPs, a default gateway, and DNS configured by hand via netplan
- `client2` created by cloning `client1`

### Problems solved

| Problem | Root cause | Fix |
|---|---|---|
| Hyper-V wouldn't enable | Windows Home doesn't support it | Switched to VirtualBox |
| Installer hung on `unattended-upgrades` | Bridged Wi-Fi gave no reliable DHCP/internet at install time | Used NAT during install, or skipped automatic updates |
| `ip_forward` reset to `0` after reboot | `systemd-networkd` resets it after `systemd-sysctl` has already applied it | Oneshot systemd service that sets it after the network is up |
| Couldn't ping the host from a VM | Windows Firewall blocked inbound ICMP on the Host-only adapter | Enabled the ICMPv4 echo rule |
| `apt update` failed with name resolution error | Static netplan config had no `nameservers` | Added DNS servers to netplan |
| `netplan apply` failed on the cloned VM | Leftover `match: macaddress` block from the original VM | Removed the `match` block |

### Concepts covered

Adapter vs router, Bridged / NAT / Internal / Host-only networking, static vs DHCP addressing, private ranges and `/24` subnets, SNAT/MASQUERADE mechanics, `netplan try` vs `apply`, `systemctl enable` vs `start`.

---

## Lab 2 — Hybrid Cloud: Site-to-Site IPsec VPN

**Goal:** Connect the Lab 1 network to a real AWS VPC over an IPsec tunnel, so lab machines can reach private cloud resources.

### Topology

```
HOME LAB                                         AWS VPC (192.168.0.0/24)
10.10.10.0/24                                    
                                                 [public subnet]
client1 ┐                                         vpn-gw-01 (Amazon Linux, Libreswan)
client2 ┼── router-vm (StrongSwan) ═══ IPsec ═══╪═   192.168.0.37 / <VPN_GW_PUBLIC_IP>
        ┘   10.10.10.1   (initiator)  IKEv2     [private subnet 192.168.0.0/27]
                                                  lab-instance-01 (no public IP)
```

- **router-vm** initiates the tunnel, so no port forwarding is needed on the home router.
- **vpn-gw-01** responds, with `left` pinned to its private address.
- Both ends use IKEv2 with a pre-shared key, AES-256 / SHA-256 / MODP2048.
- vpn-gw-01 is managed through **SSM Session Manager** (IAM role, no SSH key, no open port 22).

### AWS side

- VPC with a private subnet (`lab-instance-01`) and a public subnet (`vpn-gw-01`)
- Internet gateway attached to the VPC, public subnet routed to it
- Source/destination check disabled on `vpn-gw-01` (it forwards traffic not addressed to itself)
- Security group on `vpn-gw-01`: UDP 500 and 4500 from the home public IP, all ICMP from `10.10.10.0/24`
- IAM role with `AmazonSSMManagedInstanceCore` attached to the instances

### Configuration (sanitized)

**vpn-gw-01** (Libreswan), `/etc/ipsec.d/home-lab.conf`:

```
conn home-lab
    authby=secret
    auto=add
    ikev2=insist
    left=192.168.0.37
    leftid=@vpn-gw
    leftsubnet=192.168.0.0/24
    right=<HOME_PUBLIC_IP>
    rightid=@router-vm
    rightsubnet=10.10.10.0/24
    ike=aes256-sha256;modp2048
    phase2alg=aes256-sha256
    dpddelay=30
    dpdtimeout=120
    dpdaction=restart
```

`/etc/ipsec.d/home-lab.secrets` (mode `600`):

```
: PSK "<YOUR_LONG_RANDOM_PSK>"
```

**router-vm** (StrongSwan, `swanctl`), `/etc/swanctl/swanctl.conf`:

```
connections {
    home-lab {
        version = 2
        local_addrs = %any
        remote_addrs = <VPN_GW_PUBLIC_IP>

        local  { auth = psk; id = @router-vm }
        remote { auth = psk; id = @vpn-gw }

        children {
            home-lab {
                local_ts = 10.10.10.0/24
                remote_ts = 192.168.0.0/24
                esp_proposals = aes256-sha256
                start_action = start
            }
        }
        proposals = aes256-sha256-modp2048
    }
}

secrets {
    ike-home-lab {
        id-1 = @router-vm
        id-2 = @vpn-gw
        secret = "<YOUR_LONG_RANDOM_PSK>"
    }
}
```

**router-vm** NAT exemption (must sit above the MASQUERADE rule from Lab 1):

```bash
sudo iptables -t nat -I POSTROUTING 1 -s 10.10.10.0/24 -d 192.168.0.0/24 -j ACCEPT
sudo netfilter-persistent save
```

### Problems solved

| Symptom | Root cause | Fix |
|---|---|---|
| `dnf install strongswan` found nothing on Amazon Linux 2023 | Not in the default repos | Used Libreswan on the AWS side |
| `ipsec verify` returned "unknown command" | Removed from newer Libreswan | Used `systemctl status ipsec` and `ipsec status` |
| `PSK data malformed` in the pluto log | Passphrase not wrapped in quotes in `.secrets` | Quoted the value |
| `apt update` said Release files "not valid yet" | VM clock was about 12 days behind | Set the date and installed `systemd-timesyncd` |
| `strongswan-starter.service` not found | Ubuntu's package uses the `swanctl` / `charon-systemd` backend, so `/etc/ipsec.conf` is ignored | Rewrote the config in `swanctl.conf` format |
| `NO_PROPOSAL_CHOSEN` and "no suitable connection found with IKEv2 policy" | `left=%any` on Libreswan | Set `left=` to the instance's private IP |
| `AUTHENTICATION_FAILED` with byte-identical PSKs | Identity type mismatch: Libreswan used FQDN (`@name`), StrongSwan used a bare name | Used `@router-vm` / `@vpn-gw` on both sides |
| Ping through the tunnel sent 0 bytes | MASQUERADE rewrote the source address before IPsec could match it | NAT exemption rule above MASQUERADE |
| Tunnel "established" but no replies | Stale SAs out of sync between the two daemons | Stopped both daemons, ran `ip xfrm state flush` and `ip xfrm policy flush`, re-established the tunnel |

### Useful debugging commands

```bash
sudo swanctl --list-conns          # StrongSwan: loaded connections
sudo swanctl --list-sas            # StrongSwan: live SAs and byte counters
sudo swanctl --initiate --child home-lab
sudo ipsec status                  # Libreswan: connections and SAs
sudo journalctl -u ipsec -f        # Libreswan: live negotiation log
sudo ip xfrm policy                # kernel: which traffic gets encrypted
sudo ip xfrm state                 # kernel: installed SAs and keys
sudo iptables -t nat -L -n -v --line-numbers
sudo tcpdump -i enp0s3 udp port 4500 -n
```

### Concepts covered

IPsec/IKEv2 basics, traffic selectors, NAT traversal, tunnel encapsulation (inner vs outer packets), source/destination checks, security groups, VPC routing, SSM Session Manager and IAM roles, interoperability between StrongSwan and Libreswan, kernel XFRM policy vs daemon state.

### Status

- [x] Tunnel established and passing traffic (router-vm to vpn-gw-01)
- [ ] Verify `client1` / `client2` reach the VPC through router-vm
- [ ] Reach `lab-instance-01` through the tunnel
- [ ] Replace the hardcoded home IP with Dynamic DNS
- [ ] Attach an Elastic IP to `vpn-gw-01` so its address survives stop/start
- [ ] Tighten security groups and review IAM permissions
- [ ] Rebuild the VPC in Terraform
- [ ] Turn on VPC Flow Logs and find tunnel traffic in them

---

## Cost notes

- Public IPv4 addresses cost about $0.005/hour each (free tier covers 750 hours/month for the first 12 months).
- Only one `t2.micro`/`t3.micro` runs free around the clock, so stop instances when not in use.
- The AWS managed Site-to-Site VPN costs about $36/month per tunnel, so I used a self-managed tunnel instead.

## Security notes

- Config files in this repo use placeholders for the PSK and public IPs.
- Security group rules are scoped to a single `/32` instead of `0.0.0.0/0`.
