# Cybersecurity-Labs
Here are some labs I completed in my Masters Program
# Lab 01 — Build Your Lab

**Completed:** September 2026
**Environment:** Ubuntu 24.04.4 LTS guest · VirtualBox 7.2.16 · Windows 11 host
**Tools:** VirtualBox · ufw · OpenSSH · Docker & Compose · nmap · Wireshark · draw.io

Build a self-contained Linux lab environment from scratch, harden it against
common remote-access attacks, and use it to inventory and observe a live network.

> Hardware identifiers (MAC addresses) have been redacted from screenshots.
> See [REDACTION.md](../REDACTION.md).

---

## 1. Virtual machine

Created a VM named `cyb540-ubuntu` in VirtualBox with 8 GB RAM, 4 vCPUs, and a
60 GB dynamically allocated disk — above the suggested 4 GB / 2 vCPU / 40 GB,
since the host has 32 GB of RAM and later labs need the headroom.

The network adapter was set to **bridged** mode, which puts the VM directly on
the physical network as its own device, so the router assigns it an IP address
the same way it would a phone or laptop.

![Ubuntu desktop running in VirtualBox](screenshots/01-vm-desktop-running.png)

## 2. Updates and tooling

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server nmap git curl
```

![apt upgrade complete](screenshots/02-apt-upgrade-complete.png)

`sudo` lets a permitted user run a command with root privileges. Those commands
are logged in `/var/log/auth.log`.

```bash
ip addr
```

![ip addr showing bridged address](screenshots/03-ip-addr-bridged.png)

The VM received **192.168.1.140/24** on `enp0s3` — an address on the physical
LAN, confirming the bridge worked.

## 3. Host firewall

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

![ufw active with OpenSSH allowed](screenshots/04-ufw-status-active.png)

The allow rule comes **before** enabling the firewall. Reversing the order would
leave a window where the firewall is up and port 22 is closed — recoverable on a
VM with console access, permanent on a remote server.

## 4. Key-based SSH

```powershell
# Windows host
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh USER@VM_IP `
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

![successful key-based SSH login](screenshots/05-ssh-key-login.png)

The key is copied and tested **before** password login is disabled. Otherwise a
non-working key plus disabled passwords means no way into the machine at all.

Two changes in `/etc/ssh/sshd_config`:

| Setting | What it stops |
|---|---|
| `PasswordAuthentication no` | Logging in over SSH with a password |
| `PermitRootLogin no` | Direct SSH login as the root user |

## 5. Containers

```bash
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
docker run hello-world
```

![docker hello-world output](screenshots/06-docker-hello-world.png)

![docker compose success banner](screenshots/07-docker-compose-banner.png)

The course repository provided `docker-compose.yml` (defines the check
container), `README.md` (explains the lab), and `harden.sh` (a hardening script).

## 6. Network discovery and mapping

```bash
ip route
```

![ip route showing default gateway](screenshots/08-ip-route-gateway.png)

Default gateway is **192.168.1.1** — the router, which forwards traffic from the
local network to other networks such as the internet.

```bash
sudo nmap -sn 192.168.1.0/24
```

![nmap host discovery sweep](screenshots/09-nmap-host-sweep.png)

<!-- TODO: your report says 11 devices and lists 192.168.1.22 as the iPad, but the
     scan above shows the iPad at .33. Reconcile these before publishing — an
     employer reading closely will spot the mismatch. -->

![network topology map](screenshots/10-network-map.png)

A network map helps secure a network because it shows what is actually
connected. That makes it easier to spot unknown devices, identify possible
threats, and troubleshoot problems. The sweep surfaced two streaming devices and
one unidentified host that would otherwise have gone unnoticed.

## 7. Packet capture

```
dns.qry.name contains "example"
```

![DNS query for example.com](screenshots/11-dns-query-example-com.png)

```
tls.handshake.type == 1
```

![TLS Client Hello showing SNI](screenshots/12-tls-client-hello-sni.png)

<!-- TODO: Q11 in your report is unfinished — it names the domain but never gives
     the resolved address. Click the DNS response packet, expand Answers, and put
     the real address here before publishing. -->

DNS requests are often readable because they may be sent in plain text. After
the TLS handshake the data is encrypted, so Wireshark cannot read the actual
communication. Notably, the Client Hello still exposes the destination hostname
in the SNI field — encryption hides *what* was sent, not *where*.

## 8. Snapshot

![VirtualBox snapshot named Lab 1 complete](screenshots/13-snapshot-lab1-complete.png)

A snapshot restores the machine to a known-good state if a later lab breaks
something — the same role a backup plays.

---

## Problems and how I solved them

### Ubuntu 26.04 would not boot under VirtualBox 7.2.16

**Symptom.** Black screen after GRUB, then kernel `soft lockup - CPU stuck`
messages across multiple vCPUs.

**Investigation.** Ruled out Hyper-V interference by confirming Task Manager
still reported direct hardware virtualization. Reduced the VM from 4 vCPUs to 2
in case of an SMP timing issue — no change. Booted the safe-graphics GRUB entry
to bypass GPU negotiation — also hung.

**Root cause.** A display-driver incompatibility between the 26.04 kernel and
VirtualBox 7.2.16, which predates that release. Confirmed by the course
instructor as a known issue affecting other students on the VM path.

**Fix.** Installed Ubuntu 24.04.4 LTS instead, which booted on the first attempt.

### Host could not reach the VM over the bridged adapter

**Symptom.** `ssh` from the host timed out. `ping` host→VM returned
"Destination host unreachable" from the host's own address.

**Investigation.** Confirmed via `ip addr` that the VM's address had not changed.
Pinged in reverse, VM→host — also unreachable. Both machines held addresses in
the same `/24` and the VM had working outbound internet, so the failure was
specific to host-to-guest traffic on the local segment.

**Root cause.** Bridged mode over a USB Wi-Fi adapter. The adapter would not
pass frames for the VM's second MAC address, so ARP never resolved and no
connection could be established in either direction.

**Fix.** Added a second adapter in host-only mode, giving host and VM a private
segment that bypasses the wireless bridge. The bridged adapter was retained so
the VM keeps its LAN address for the discovery portion of the lab.

---

## What I learned

<!-- TODO: three or four takeaways in your own words. Candidates:
     - ordering discipline in firewall and SSH changes (open before enable,
       test the key before disabling passwords)
     - encryption hides content but not destination
     - what a network inventory actually reveals about a network you thought
       you knew
     - isolating a failure across host, hypervisor, and guest layers -->

## References

- [Ubuntu Server Guide — OpenSSH](https://ubuntu.com/server/docs/service-openssh)
- [ufw manual](https://manpages.ubuntu.com/manpages/noble/man8/ufw.8.html)
- [Nmap host discovery](https://nmap.org/book/man-host-discovery.html)
- [Wireshark display filters](https://www.wireshark.org/docs/dfref/)
