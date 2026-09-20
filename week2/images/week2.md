# PENETRATION TESTING REPORT
## Footprinting (theHarvester) & Network Scanning (Zenmap) — Blue Team Perspective
**W2-PM-FINAL | Cybersecurity & Ethical Hacking | NetworkWalks**

| | |
|---|---|
| **Pentester name** | Khadija |
| **Program / Batch** | NetworkWalks Cybersecurity & Ethical Hacking, Batch 83 |
| **Date** | 20 September 2026 |
| **Modules completed** | W2-PM4 (theHarvester footprinting), W2-PM5 (Zenmap network scanning) |
| **Targets** | 1. `microsoft.com` (passive OSINT only, as specified by the module)<br>2. My own isolated lab network (VirtualBox NAT Network `10.0.0.0/24`) |
| **Permission** | theHarvester only reads public data sources and never contacts the target. Network scanning was done only on my own lab. |
| **Phases covered** | Phase 1: Reconnaissance & Footprinting<br>Phase 2: Scanning & Network Discovery |

---

## 1. Liability Disclaimer

I performed these activities only on systems I own (my own lab) or through passive public-source queries. These materials are for education and research purposes only. Do not use anything from here to break the law. Unauthorised access is a crime in most countries, even when nothing is damaged.

---

## 2. Introduction

This report covers two Week 2 modules. In **W2-PM4**, I used theHarvester to gather emails and hosts related to `microsoft.com` from public sources. In **W2-PM5**, I used Nmap/Zenmap to discover live hosts on my own lab network. Together they show how an attacker moves from collecting public information (passive, undetectable by the target) to mapping live machines (active, detectable).

I wrote this report from a **blue team perspective**: for each step, I explain what an attacker learns and how a defender can reduce or detect it.

Every step includes the command used, the observed result, and the evidence.

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux (VirtualBox VM) | Platform for reconnaissance and scanning |
| theHarvester 4.10.1 | Gather emails, hosts and IPs from public sources (passive OSINT) |
| Nmap / Zenmap 7.99 | Discover live hosts, IP and MAC addresses; draw a topology |
| `tee` | Save command output to a text file while displaying it |
| `ip a` | Identify the Kali VM's IP and MAC address |
| VirtualBox NAT Network | Isolated lab network (`10.0.0.0/24`) |
| Windows + Zenmap 7.991 | Supplementary test on the host (see section 4.2.4) |

---

## 4. Activities Performed

### 4.1 Footprinting with theHarvester (W2-PM4)

theHarvester queries public sources (search engines, certificate databases, leak databases) and extracts emails, hostnames and IPs. It does not contact the target's servers.

#### Task 1: Baidu source, limit 1000
```bash
theHarvester -d microsoft.com -l 1000 -b baidu | tee task1_baidu.txt
```
**Result:**
- First run: **8 emails** and **18 hosts** found.
- Second run (same command): **no results**.

Emails from the first run included `info@`, `contactspocaccess@`, `viva-noreply@` and some placeholder or malformed entries (`abc@`, `someone@`, `user@contoso.onmicrosoft.com`, and entries starting with a comma such as `,contact@`). Hosts included `account`, `appsource`, `developer`, `docs`, `graph`, `learn`, `msdn` and `msrc` under `microsoft.com`.

**Observations:**
- Results varied between two identical runs. A likely reason is that the source limits or blocks repeated automated requests, but I did not verify the cause.
- The raw output is noisy (placeholders, extraction errors), so an analyst must clean and verify results.

![Task 1, first run](week2/images/task1_baidu_run1.png)
![Task 1, second run](week2/images/task1_baidu_run2.png)

#### Task 2: All sources, limit 50
```bash
theHarvester -d microsoft.com -l 50 -b all 2>&1 | tee task2_all.txt
```
**Result:**
- **3 emails:** `dotnet-docker-bot@`, `opencode@`, `secure@` (bot or service addresses; `secure@` is the usual security reporting address).
- **1364 hosts**, some with resolved IPs (format `host:IP`).
- Many sources returned `Missing API key` errors (for example bevigil, bitbucket, builtwith, brave, securityscorecard). This is expected because those sources need keys; the free sources still returned results.
- The Hudson Rock source reported 43 hosts and, according to that third-party source, about 602,741 compromised credentials linked to the domain, including about 16,148 employees. I did not verify these figures.

**Observations:**
- Using all sources returned far more results (1364 hosts) than Baidu alone (18). Different sources return different data, so several should always be used.
- Some hostnames look internal (for example `*.redmond.corp.microsoft.com`, `*.sourcedepot.corp.microsoft.com`).
- Some names look like test or development environments (`wwwbeta`, `wwwqa`, `*.dev.insights`).
- Hundreds of `fabric.microsoft.com` subdomains show how one cloud product can create many hostnames to inventory.
- Some entries such as `2Fblogs.microsoft.com` are extraction errors (`%2F` is an encoded `/`).

![Task 2, start of run](week2/images/task2_all_start.png)
![Task 2, emails and hosts](week2/images/task2_all_emails_hosts.png)
![Task 2, end of run](week2/images/task2_all_end.png)

**What an attacker learns:** emails for phishing, hostnames for finding forgotten or weaker services, and naming conventions that reveal how the infrastructure is organized.

**What a defender does with it:** run the same tool on their own domain, then remove exposed or stale entries.

---

### 4.2 Network Scanning with Zenmap (W2-PM5)

A **ping scan** (`nmap -sn`) only asks each address "are you there?". It does not scan ports.

#### 4.2.1 Local IP and subnet
```bash
ip a
```
- Kali IP: `10.0.0.2/24` on `eth0`
- Kali MAC: `08:00:27:5a:87:bc` (VirtualBox prefix `08:00:27`)
- Subnet: `10.0.0.0/24` (mask `255.255.255.0`)

![ip a output](week2/images/ip_a_kali.png)

#### 4.2.2 Discover live hosts
Zenmap: Target `10.0.0.0/24`, Profile **Ping scan**, command `nmap -sn 10.0.0.0/24`.

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-20 15:02 -0400
Nmap scan report for 10.0.0.1
Host is up (0.00099s latency).
MAC Address: 52:54:00:12:35:00 (QEMU virtual NIC)
Nmap scan report for 10.0.0.3
Host is up (0.00068s latency).
MAC Address: 52:54:00:12:35:00 (QEMU virtual NIC)
Nmap scan report for 10.0.0.2
Host is up.
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.94 seconds
```

| IP | Role | MAC |
|---|---|---|
| `10.0.0.1` | Virtual gateway of the VirtualBox NAT Network | `52:54:00:12:35:00` |
| `10.0.0.2` | My Kali VM (the scanning machine) | `08:00:27:5a:87:bc` (from `ip a`; Nmap does not list the scanner's own MAC) |
| `10.0.0.3` | Probably a virtual network service (for example DHCP); not verified | `52:54:00:12:35:00` |

**Observations:**
- **3 live hosts**, including my own VM.
- `10.0.0.1` and `10.0.0.3` share the same MAC. Here this is consistent with both being served by VirtualBox's virtual network engine. On a real network, two IPs sharing one MAC can indicate ARP spoofing or a multi-homed device, so the cause must be checked before drawing conclusions.

![Zenmap ping scan](week2/images/zenmap_ping_scan.png)

#### 4.2.3 Topology (Task 7)
In Zenmap: Topology tab, Legend on, Save Graphic as PDF (`zenmap_topology.pdf`).

- `localhost` (black, center) is the Kali VM; `10.0.0.1`, `10.0.0.2` and `10.0.0.3` are the discovered hosts.
- Dashed lines mean "no traceroute information". A ping scan does not run traceroute, so these lines do not show measured connections.
- Green means "fewer than 3 open ports", but no port scan was performed, so the colour carries no security meaning here.
- `10.0.0.2` is the Kali VM itself (also shown as `localhost`).

![Topology with legend](week2/images/zenmap_topology.png)

The exported file is `week2/zenmap_topology.pdf`.

#### 4.2.4 Supplementary observation: network isolation
I first ran the same scan from my Windows host. It found only `10.0.0.1` (the host's VMware VMnet8 adapter) and `10.0.0.254` (a VMware service, MAC prefix `00:50:56`), and no VirtualBox machine. The reason is that the VirtualBox NAT Network is isolated: the host has no adapter in it, so the host cannot reach the lab VMs. The VMware network and the VirtualBox network use the same `10.0.0.0/24` range but are two separate virtual networks.

I therefore ran the scan from inside the lab (Kali). Isolation of a test lab from the host and from real networks is a good practice.

---

### 4.3 Detection observation (blue team)

> **[TO COMPLETE — keep this section only if you run the capture, otherwise delete it]**
> In one Kali terminal run `sudo tcpdump -i eth0 -n arp or icmp | tee capture_scan.txt`, then run `nmap -sn 10.0.0.0/24` in another terminal and stop the capture with Ctrl+C. Describe what you actually see (for example a burst of ARP "who-has" requests to consecutive addresses) and add the screenshot below.

![tcpdump during scan](week2/images/tcpdump_scan.png)

---

## 5. Risk Analysis / Impact

These are **observations**, not confirmed vulnerabilities. No exploitation or vulnerability testing was performed. The risk levels are my own assessment for this exercise.

| # | Finding | Evidence | Potential impact | Risk level |
|---|---|---|---|---|
| 1 | Many public hostnames (1364) | theHarvester Task 2 | A large attack surface that is hard to inventory; forgotten hosts may go unpatched | Medium |
| 2 | Internal-looking hostnames visible (`*.corp.*`) | theHarvester Task 2 | Reveals naming conventions and internal infrastructure layout | Medium |
| 3 | Test/dev hostnames visible (`wwwqa`, `wwwbeta`) | theHarvester Task 2 | Non-production environments are often less protected | Medium |
| 4 | Emails exposed publicly | Task 1 and Task 2 | Targets for phishing; most found were generic, bot or placeholder addresses | Low |
| 5 | Third-party report of compromised credentials | Hudson Rock source (unverified) | If accurate, raises risk of credential reuse and targeted phishing | Medium |
| 6 | Live hosts discoverable by ping scan | Zenmap, 3 hosts | Any device on the segment can map the network; unknown devices could go unnoticed | Low |
| 7 | Two IPs sharing one MAC | Zenmap, `10.0.0.1` and `10.0.0.3` | Explained here by virtual infrastructure, but could indicate spoofing on a real network | Low |

---

## 6. Recommendations

1. **Run theHarvester on your own domain regularly** to see what an attacker sees, using several sources.
2. **Keep an inventory of subdomains** and remove DNS records for services that no longer exist.
3. **Do not expose test and development environments** publicly, or protect them like production.
4. **Avoid leaking internal naming** in public certificates, DNS records and code repositories.
5. **Reduce email exposure:** prefer role-based addresses, add phishing training, and enable MFA so exposed emails are harder to exploit.
6. **Monitor for credential leaks** and force password resets when compromised credentials are reported.
7. **Scan your own network periodically** to keep a device inventory, and investigate any unknown host.
8. **Watch for scanning activity:** a burst of ARP or ICMP requests to consecutive addresses is a typical sign of a network sweep. Log it and alert on it.
9. **Check unusual ARP or MAC behaviour** (for example duplicate MACs) before concluding it is harmless.
10. **Scan only with authorization**, and keep lab networks isolated from real ones.

---

## 7. Conclusion

In Week 2, I completed one footprinting module and one scanning module. With theHarvester, public sources alone gave 8 emails and 18 hosts using Baidu, and 1364 hosts and 3 emails using all sources, without touching the target. With Zenmap, a ping scan of my isolated lab found 3 live hosts, and I learned why a scan from the Windows host could not see the VirtualBox network.

Main lessons:
- Passive reconnaissance reveals a lot and is undetectable by the target, so defenders must check their own exposure.
- Results differ between sources and between runs, and raw output must be verified and cleaned.
- Scan results depend on where the scan runs from and on how the network is isolated.
- Findings should be reported as observations, with the risk and the fix, not as proven vulnerabilities.
- All scanning must stay within an authorized scope.

---

## 8. Evidence Collected

Files are stored in `weeks/week2/`:

| File | Description |
|---|---|
| `theharvester/task1_baidu.txt` | Output of Task 1 |
| `theharvester/task2_all.txt` | Output of Task 2 |
| `zenmap/zenmap_scan.txt` | Nmap output from Kali |
| `zenmap_topology.pdf` | Topology exported from Zenmap |
| `images/*.png` | Screenshots referenced above |

---

**Author:** Khadija — Cybersecurity & Ethical Hacking Internship, NetworkWalks, Batch 83
**Project:** Week 02 | **Repository:** `NetworkWalks-Cybersecurity-EthicalHacking-Batch83`
