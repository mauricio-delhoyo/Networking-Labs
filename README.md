# Networking & Cloud Labs

Hands-on labs for learning Linux, networking, and cloud fundamentals, built toward becoming a stronger cloud engineer. Each lab builds on the previous one, and every lab includes the problems I hit and how I fixed them, since the debugging is where most of the learning happened.

| Lab | Topic | Stack |
|---|---|---|
| [Lab 1](#lab-1--linux-router-with-nat) | Linux router (NAT + forwarding) between a home network and an isolated lab network | VirtualBox, Ubuntu Server, netplan, iptables |
| [Lab 2](#lab-2--hybrid-cloud-site-to-site-ipsec-vpn) | Site-to-site IPsec VPN between the home lab and an AWS VPC, reaching a private EC2 instance through it | AWS (VPC, EC2, SSM), StrongSwan, Libreswan |

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

**Goal:** Connect the Lab 1 network to a real AWS VPC over an IPsec tunnel, so lab machines can reach private cloud resources, and finish by logging in to an EC2 instance that has no public IP.

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
- lab-instance-01 has no internet path, so SSM can't reach it. I log in to it with an SSH key through the tunnel instead.

### AWS side

- VPC with a private subnet (`lab-instance-01`) and a public subnet (`vpn-gw-01`)
- Internet gateway attached to the VPC, public subnet routed to it
- Source/destination check disabled on `vpn-gw-01` (it forwards traffic not addressed to itself)
- Security group on `vpn-gw-01`: UDP 500 and 4500 from the home public IP, all ICMP from `10.10.10.0/24`. The UDP 500 and 4500 rules are separate, so both must be updated when the home IP changes.
- IAM role with `AmazonSSMManagedInstanceCore` attached to `vpn-gw-01`

**What reaching `lab-instance-01` through the tunnel needs:**

- IP forwarding on `vpn-gw-01` (`net.ipv4.ip_forward = 1`), so it passes tunnel traffic on to the private subnet
- A route in the private subnet's route table: destination `10.10.10.0/24`, target `vpn-gw-01`, so replies find their way back into the tunnel
- Security group on `lab-instance-01`: inbound all ICMP and SSH (22) from `10.10.10.0/24`
- The instance's `.pem` key on the machine I connect from

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

### Starting the VPN each session

I stop both instances at the end of each day. EC2 assigns a new public IP to `vpn-gw-01` on every start, so the router-vm config needs the new address each time.

**1. In the EC2 console:** start `vpn-gw-01` and `lab-instance-01`, and copy `vpn-gw-01`'s new **Public IPv4 address**.

**2. On vpn-gw-01 (Session Manager):** check the VPN service is running. It starts at boot.

```bash
sudo systemctl status ipsec        # should say: active (running)
```

**3. On router-vm:** point the config at the new address, then restart StrongSwan.

```bash
sudo sed -i 's/remote_addrs = .*/remote_addrs = NEW_PUBLIC_IP/' /etc/swanctl/swanctl.conf   # replace NEW_PUBLIC_IP with the real address
sudo grep remote_addrs /etc/swanctl/swanctl.conf      # verify the change (sed prints nothing if it matched nothing)
sudo systemctl stop strongswan
sudo ip xfrm state flush           # clear leftover tunnel keys and rules from the previous day
sudo ip xfrm policy flush
sudo systemctl start strongswan
sudo swanctl --load-all            # read the config into the running daemon
```

The config has `start_action = start`, so the tunnel dials by itself when StrongSwan starts. Running `sudo swanctl --initiate --child home-lab` as well just reports `not establishing CHILD_SA ... due to existing duplicate`, which means the tunnel is already up.

**4. Check it:**

```bash
sudo swanctl --list-sas                        # want: ESTABLISHED, with the new address as remote
ping -I 10.10.10.1 -c 4 192.168.0.37           # from router-vm; or ping 192.168.0.37 from client1
```

**5. If it doesn't come up,** the usual cause is the home IP. Compare `curl -4 ifconfig.me` (run on router-vm) with the source on `vpn-gw-01`'s security group, and update **both** the UDP 500 and UDP 4500 rules if it changed.

### Reaching lab-instance-01

From router-vm, with the tunnel up. The `-b` flag sets the source address to `10.10.10.1`, which is the address the tunnel carries. Without it, router-vm's own traffic leaves with its home address and never enters the tunnel.

```bash
ssh -b 10.10.10.1 -i ~/lab-instance-01-key.pem ec2-user@<LAB_INSTANCE_PRIVATE_IP>
```

If you can't connect, test the path first with `ping -I 10.10.10.1 -c 4 <LAB_INSTANCE_PRIVATE_IP>`:

| Result | Likely cause |
|---|---|
| No replies, nothing shows up in `sudo tcpdump -i enX0 icmp -n` on vpn-gw-01 | Traffic isn't reaching vpn-gw-01: tunnel down, or `ip_forward` is `0` (check `sudo sysctl net.ipv4.ip_forward`; it can reset after a restart, as in Lab 1) |
| Requests leave vpn-gw-01, no replies | Missing return route in the private subnet's route table, or lab-instance-01's security group is missing the ICMP/SSH rules |
| Ping works, SSH times out | Security group is missing the SSH (22) rule from `10.10.10.0/24` |
| `Permission denied (publickey)` | Wrong key or wrong username (`ec2-user` on Amazon Linux) |

Copying the key to router-vm: run `scp` from the machine that has the file (for me, PowerShell on Windows). `No route to host` usually means the target VM is powered off.

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

### Troubleshooting after a break

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
sudo sysctl net.ipv4.ip_forward    # is forwarding on? (needs to be 1 on both gateways)
sudo iptables -t nat -L -n -v --line-numbers
sudo tcpdump -i enp0s3 udp port 4500 -n    # router-vm: tunnel packets leaving
sudo tcpdump -i enX0 icmp -n               # vpn-gw-01: pings arriving from the tunnel
```

### Concepts covered

IPsec/IKEv2 basics, traffic selectors, NAT traversal, tunnel encapsulation (inner vs outer packets), source/destination checks, security groups, VPC routing, SSM Session Manager and IAM roles, interoperability between StrongSwan and Libreswan, kernel XFRM policy vs daemon state.

Also: a tunnel is shared state held in memory on both ends, so one side can forget it while the other still shows it as up; a config file vs the state a service actually loaded; what a PSK does (authenticates the peers) and what it does not do (encrypt traffic); dynamic home IPs and why Dynamic DNS does not help when security groups only accept IP ranges and the home side always initiates; using an EC2 instance as a gateway (IP forwarding, source/destination check, a VPC route pointing at an instance); choosing the source address of a connection (`ping -I`, `ssh -b`) so it matches a tunnel's traffic selector; key-based SSH to a private instance; AWS public IPv4 billing.

### Status

- [x] Tunnel established and passing traffic (router-vm to vpn-gw-01)
- [x] `client1` reaches the VPC through router-vm (`client2` not tested)
- [x] Reached `lab-instance-01` (no public IP) over SSH through the tunnel
- [x] Stopped hardcoding the home IP in Libreswan (`right=%any`)
- [x] Rotated the PSK after it appeared in a chat log

---

## Cost notes

- Public IPv4 addresses cost about $0.005/hour each (about $3.65/month). The free tier only covers them during an account's first 12 months, and only for EC2.
- This lab runs in an account older than 12 months that belongs to an AWS Organization, so no free tier applies (the Billing Free Tier page is empty). Rough costs, which vary by region: about $8.50/month per `t2.micro` running 24/7, about $3.65/month for the public IP, and about $0.65/month per 8 GB disk. Stop instances when not in use.
- An auto-assigned public IP is released when the instance stops, so it costs nothing while stopped. A new one is assigned on the next start, which is why `remote_addrs` is updated every session.
- The AWS managed Site-to-Site VPN costs about $36/month per tunnel, so I used a self-managed tunnel instead.
- Billing data lags by hours, so a budget alert warns after the fact. Set a low threshold.

## Security notes

- Never commit PSKs, private keys, or `.pem` files. Keep them out of the repo folder entirely.
- Config files in this repo use placeholders for the PSK and public IPs.
- Treat any secret that appears in a log, chat, or screenshot as exposed, and rotate it.
- Security group rules are scoped to a single `/32` (home IP) or the lab subnet (`10.10.10.0/24`) instead of `0.0.0.0/0`.
- The private instance's `.pem` key lives only on router-vm. Delete it when the lab ends.
