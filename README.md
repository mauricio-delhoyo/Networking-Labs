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
- **vpn-gw-01** responds, with `left` pinned to its private address and `right=%any` so the home IP can change.
- Both ends use IKEv2 with a pre-shared key, AES-256 / SHA-256 / MODP2048.
- vpn-gw-01 is managed through **SSM Session Manager** (IAM role, no SSH key, no open port 22).

### AWS side

- VPC with a private subnet (`lab-instance-01`) and a public subnet (`vpn-gw-01`)
- Internet gateway attached to the VPC, public subnet routed to it
- Source/destination check disabled on `vpn-gw-01` (it forwards traffic not addressed to itself)
- Security group on `vpn-gw-01`: UDP 500 and 4500 from the home public IP, all ICMP from `10.10.10.0/24`. The UDP 500 and 4500 rules are separate, so both must be updated when the home IP changes.
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
    right=%any
    rightid=@router-vm
    rightsubnet=10.10.10.0/24
    ike=aes256-sha256;modp2048
    phase2alg=aes256-sha256
    dpddelay=30
    dpdtimeout=120
    dpdaction=restart
```

`right=%any` accepts the tunnel from whatever public address the home connection has, so a changing ISP address no longer needs a config edit here. The peer still has to identify as `@router-vm` and prove it knows the PSK. Libreswan only reads this file when the service starts, so run `sudo systemctl restart ipsec` after any edit.

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

### Resuming the lab after a break (Oct 2026)

Coming back to the lab after a few weeks, the tunnel was down, and it took several rounds to find out why. Each of these looked like a different problem but came from the same two facts: the home IP changes, and EC2 gives a new public IP on every stop/start.

| Symptom | Root cause | Fix |
|---|---|---|
| Dial hangs, then `peer not responding` after 5 retransmits | vpn-gw-01's public IP changed after a stop/start, or the home IP changed and the security group still allowed the old one | Updated `remote_addrs` on router-vm and the security group sources |
| Handshake gets a reply on port 500, then retransmits forever on port 4500 | Only the UDP 500 rule had the new home IP. The handshake moves to UDP 4500 once NAT is detected, and that is a separate rule | Updated both the UDP 500 and 4500 rules |
| Fast `NO_PROPOSAL_CHOSEN` after editing Libreswan's `right=` | Libreswan reads its `.conf` and `.secrets` at start. The running service still held the old definition (`ipsec status` showed the old IP) | `sudo systemctl restart ipsec` |
| `ESTABLISHED` on router-vm, pings fail, `ip xfrm state` empty on vpn-gw-01 | vpn-gw-01 built the tunnel, its liveness probes got no answer, and it deleted the tunnel about 90 seconds later. router-vm never noticed | Stopped StrongSwan on router-vm, flushed xfrm state and policy, started it again so a fresh tunnel formed. The tunnel then stayed up. Root cause not confirmed; suspected leftover state from earlier failed attempts or a changed NAT mapping |
| `remote_addrs = NEW_IP` in `swanctl.conf` | A placeholder was pasted literally. `sed` does not report an error when a pattern matches nothing, so later substitutions silently did nothing | Re-ran the substitution on the text actually in the file, and used `grep` to verify |

**Debug order that worked:**

1. Is there an `ESTABLISHED` tunnel on router-vm? (`swanctl --list-sas`)
2. Are bytes leaving (`out`), and is anything coming back (`in`)? `out` stuck at 0 means the problem is on router-vm (NAT exemption, forwarding). `out` rising with `in` at 0 means the AWS side or lost state.
3. What does the other end say? (`journalctl -u ipsec -f`, `ipsec status`, `ip -s xfrm state` on vpn-gw-01)
4. Does the running service match the file on disk? (A config file edit does nothing until the service is restarted or reloaded.)

### Useful debugging commands

```bash
sudo swanctl --list-conns          # StrongSwan: loaded connections
sudo swanctl --list-sas            # StrongSwan: live SAs and byte counters
sudo swanctl --initiate --child home-lab
sudo swanctl --terminate --ike home-lab   # close router-vm's side of the tunnel
sudo ipsec status                  # Libreswan: connections and SAs
sudo journalctl -u ipsec -f        # Libreswan: live negotiation log
sudo ip xfrm policy                # kernel: which traffic gets encrypted
sudo ip xfrm state                 # kernel: installed SAs and keys
sudo ip -s xfrm state              # same, with packet counters
cat /proc/net/xfrm_stat            # kernel: why IPsec packets were dropped
sudo iptables -t nat -L -n -v --line-numbers
sudo tcpdump -i enp0s3 udp port 4500 -n
```

### Concepts covered

IPsec/IKEv2 basics, traffic selectors, NAT traversal, tunnel encapsulation (inner vs outer packets), source/destination checks, security groups, VPC routing, SSM Session Manager and IAM roles, interoperability between StrongSwan and Libreswan, kernel XFRM policy vs daemon state.

Also: a tunnel is shared state held in memory on both ends, so one side can forget it while the other still shows it as up; a config file vs the state a service actually loaded; what a PSK does (authenticates the peers) and what it does not do (encrypt traffic); dynamic home IPs, and why Dynamic DNS does not help when security groups only accept IP ranges and the home side always initiates; AWS public IPv4 billing.

### Status

- [x] Tunnel established and passing traffic (router-vm to vpn-gw-01)
- [x] `client1` reaches the VPC through router-vm (`client2` not tested yet)
- [x] Stop hardcoding the home IP in Libreswan (`right=%any`)
- [x] Rotate the PSK after it appeared in a chat log
- [ ] Confirm the tunnel survives its first rekey (about an hour) without dropping
- [ ] Consider `dpdaction=clear` on vpn-gw-01 (with `right=%any`, `restart` has no address to dial; untested)
- [ ] Auto-update the security group's UDP 500 and 4500 rules when the home IP changes (scoped IAM user + script on router-vm)
- [ ] Attach an Elastic IP to `vpn-gw-01` so its address survives stop/start
- [ ] Reach `lab-instance-01` through the tunnel
- [ ] Tighten security groups and review IAM permissions
- [ ] Rebuild the VPC in Terraform
- [ ] Turn on VPC Flow Logs and find tunnel traffic in them

---

## Cost notes

- Public IPv4 addresses cost about $0.005/hour each (about $3.65/month), whether auto-assigned or Elastic. The free tier only covers them during an account's first 12 months, and only for EC2.
- This lab runs in an account older than 12 months that belongs to an AWS Organization, so no free tier applies (the Billing Free Tier page is empty). Rough costs, which vary by region: about $8.50/month per `t2.micro` running 24/7, about $3.65/month for the public IP, and about $0.65/month per 8 GB disk. Stop instances when not in use.
- An auto-assigned public IP is released when the instance stops, so it costs nothing while stopped but changes on the next start. An Elastic IP keeps the address but keeps billing while the instance is stopped.
- The AWS managed Site-to-Site VPN costs about $36/month per tunnel, so I used a self-managed tunnel instead.
- Billing data lags by hours, so a budget alert warns after the fact. Set a low threshold.

## Security notes

- Never commit PSKs, private keys, or `.pem` files. Keep them out of the repo folder entirely.
- Config files in this repo use placeholders for the PSK and public IPs.
- Treat any secret that appears in a log, chat, or screenshot as exposed, and rotate it.
- Security group rules are scoped to a single `/32` instead of `0.0.0.0/0`.
- If router-vm gets AWS credentials for an auto-update script, scope the IAM policy to one security group and the two actions needed to add and remove its rules.
