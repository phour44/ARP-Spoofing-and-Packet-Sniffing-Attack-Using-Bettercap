# ARP Spoofing and Packet Sniffing Attack Using Bettercap

**A Man-in-the-Middle (MITM) Lab Demonstration on a Controlled Network**

---

## Author

| Field | Details |
|---|---|
| Name | Ibrahim Ayomide Fayomi |
| Role | Cybersecurity Analyst |
| Email | fayomiibrahim44@gmail.com |
| LinkedIn | [linkedin.com/in/fayomi31](https://www.linkedin.com/in/fayomi31/) |

---

## Project Overview

This project demonstrates how an attacker positioned inside a local network can intercept all traffic flowing through a target machine using ARP spoofing and packet sniffing. The core question this project answers is: **can someone see what you are doing on the internet?** The answer is yes, if they are on the same network or have found a way in.

Using Bettercap on Kali Linux, this lab shows how a Man-in-the-Middle attack works from setup to live traffic interception. The victim machine (Windows 10) browsed real websites including Google, Yahoo, Tesla, and Facebook. The attacker captured DNS queries, HTTPS Server Name Indication (SNI) data, and plaintext HTTP responses in real time, without the victim knowing.

This project is executed strictly in a controlled, isolated lab environment for ethical cybersecurity learning and portfolio demonstration purposes only.

---

## Objectives

- Launch Bettercap on the attacker machine and explore its module system
- Use `net.probe` and `net.recon` to discover all live hosts on the network
- Identify the target device (172.20.10.6) from the discovered ARP table
- Perform ARP poisoning using `arp.spoof` to position the attacker between the victim and the gateway
- Activate `net.sniff` to capture live network traffic from the target
- Demonstrate the privacy and security risk of unencrypted traffic and TLS SNI leakage
- Show real-world impact: DNS queries, HTTPS domain exposure, and HTTP response interception

---

## Lab Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (Desktop terminal) |
| Victim OS | Windows 10 running in Oracle VirtualBox |
| Network Range | 172.20.10.0/28 |
| Gateway | 172.20.10.3 |
| Target IP | 172.20.10.6 (Windows victim machine) |
| Other Devices | 172.20.10.2, 172.20.10.4 (HR-PC), 172.20.10.5 |
| Tool | Bettercap v2.41.5 |
| Interface | eth0 |

---

## Tools and Technologies

| Tool | Purpose |
|---|---|
| Bettercap v2.41.5 | Main framework for MITM, ARP spoofing, and packet sniffing |
| net.probe | Actively discovers live hosts on the subnet |
| net.recon | Passive background recon, auto-started by net.probe |
| arp.spoof | Sends forged ARP replies to poison the ARP cache of the target and gateway |
| net.sniff | Captures and displays live network packets from intercepted traffic |
| Oracle VirtualBox | Hosts the Windows 10 victim machine |
| Kali Linux | Attacker operating system |

---

## Architecture

```
[Windows 10 Victim - 172.20.10.6]
           |
           | (ARP cache poisoned - thinks attacker is the gateway)
           |
[Kali Linux Attacker - Bettercap]  <-- traffic intercepted here
           |
           | (ARP cache poisoned - gateway thinks attacker is the victim)
           |
[Gateway - 172.20.10.3]
           |
        [Internet]
```

When ARP spoofing is active, the victim's traffic travels through the attacker machine before reaching the gateway. The attacker reads every packet in transit.

---

## Step-by-Step Methodology

### Step 1: Launch Bettercap and Explore the Manual

The first step is opening a terminal on the Kali Linux machine and launching Bettercap with the eth0 network interface.

```bash
sudo bettercap -iface eth0
```

After authentication, Bettercap starts and detects the network range 172.20.10.0/28. The `help` command opens the built-in manual, which lists all available commands including `set`, `get`, `help MODULE`, `active`, and `quit`.

**What this does:** The `-iface eth0` flag tells Bettercap which network interface to use. Kali Linux uses eth0 as the wired Ethernet interface. The tool automatically identifies the gateway and subnet.

![STARTING BETTERCAP AND OPENING ITS MANUAL BY TYPING HELP](https://github.com/user-attachments/assets/c5843a9a-32ba-4c31-a8d5-96a617d1a3b2)


---

### Step 2: Review Available Modules

Typing `help` without arguments shows all available Bettercap modules, all listed as "not running." The modules used in this project are:

- `arp.spoof` for MITM positioning
- `net.probe` for active host discovery
- `net.recon` for passive recon (auto-starts with net.probe)
- `net.sniff` for live traffic capture

![BETTERCAP DIFFERENT MODULES THAT CAN BE USED FOR DIFFERENT ATTACK _ I WILL BE USING NETPROBE_ARPSPOOF AND NETSNIFF](https://github.com/user-attachments/assets/a3065c7e-0e47-418e-83ca-d1fe9fe97a77)


---

### Step 3: Discover Live Hosts with net.probe

The `help net.probe` command shows the module documentation. Then `net.probe on` is executed to start active scanning.

```bash
help net.probe
net.probe on
```

**What this does:** `net.probe` sends dummy UDP packets to every possible IP on the subnet (172.20.10.0/28 = 16 addresses). Any host that responds is logged as a live endpoint. The `net.recon` module starts automatically alongside it to handle passive ARP-based discovery.

**Output observed:** Bettercap immediately detected three live hosts: 172.20.10.2 (Intel Corporate), 172.20.10.4, and 172.20.10.5 (both PCS Systemtechnik GmbH). IPv6 link-local addresses were also detected.

![USING NET PROBE TOOL](https://github.com/user-attachments/assets/5dbd5dee-dfb1-4f7c-a27f-0330e33fa85c)


---

### Step 4: View the ARP Table and Identify the Target

The `net.show` command prints a formatted ARP table of all discovered devices.

```bash
net.show
```

**ARP Table Result:**

| IP | MAC | Vendor | Notes |
|---|---|---|---|
| 172.20.10.3 | 08:00:27:ad:c7:68 | PCS Systemtechnik GmbH | Gateway (eth0) |
| 172.20.10.2 | 08:71:90:ff:8b:97 | Intel Corporate | Second device |
| 172.20.10.4 | de:40:2e:f5:f4:05 | PCS Systemtechnik GmbH | HR-PC |
| 172.20.10.6 | 08:00:27:85:1d:78 | PCS Systemtechnik GmbH | **Target - Windows 10** |

The target is confirmed as **172.20.10.6**. The `help arp.spoof` command is then run to review the ARP spoofing module options.

![USING NET SHOW TO SHOW ARP TABLE AND STARTING ARP SPOOF MANUAL](https://github.com/user-attachments/assets/96cbac25-c688-48d7-91aa-9ec90c019b03)


---

### Step 5: Configure ARP Spoofing and Start the MITM Attack

Two parameters are configured before activating ARP spoofing:

```bash
set arp.spoof.fullduplex true
set arp.spoof.targets 172.20.10.6
arp.spoof on
```

**Parameter breakdown:**

- `arp.spoof.fullduplex true`: Poisons both the target machine and the gateway simultaneously. This ensures traffic flows in both directions through the attacker. Without this, only one direction is captured.
- `arp.spoof.targets 172.20.10.6`: Limits the attack to only the Windows victim machine instead of the entire subnet. This is focused and controlled.
- `arp.spoof on`: Starts sending forged ARP replies. The victim now believes the attacker's MAC address belongs to the gateway, and the gateway believes the attacker's MAC belongs to the victim.

Bettercap confirms: "full duplex spoofing enabled... starting, probing 1 targets."

Then packet sniffing is started immediately:

```bash
net.sniff on
```

The first intercepted packets appear within seconds, showing HTTP requests from 172.20.10.6 to an external server.

![STARTI~1](https://github.com/user-attachments/assets/eec9ad05-b59a-40c4-b0ee-6dab65ee3beb)


---

### Step 6: Live Traffic Interception

With the MITM attack active, all traffic from 172.20.10.6 passes through the attacker machine. The terminal fills with captured data in real time.

**Key traffic types captured:**

**DNS queries** (net.sniff.dns): Every domain the victim resolves is visible, including gmail.com, yahoo.com, facebook.com, tesla.com, edge.microsoft.com, and more. DNS is unencrypted by default, so full query and response details are exposed.

**HTTPS SNI leakage** (net.sniff.https): Even though HTTPS encrypts the content of web traffic, the TLS handshake includes a Server Name Indication (SNI) field that reveals which domain is being connected to. This appears in plaintext. Examples captured: `172.20.10.6 > https://accounts.google.com`, `172.20.10.6 > https://yahoo.com`, `172.20.10.6 > https://tesla.com`.

**HTTP plaintext responses** (net.sniff.http.response): Unencrypted HTTP responses are fully captured. Microsoft's NCSI check (`msftncsi.com/ncsi.txt`) returned `HTTP/1.1 200 OK` with the full response header, including `Cache-Control: max-age=30, must-revalidate`.

**Windows Update traffic**: The HR-PC (172.20.10.4) was downloading a Windows update binary (`au.download.windowsupdate.com`), which was also visible in the sniff output.

![PACKET SNIFFING ATTACK ON GOING](https://github.com/user-attachments/assets/bdb75a27-b6b9-46b6-9661-49012ae40777)
![PACKET SNIFFING AGAINST TARGET MACHINE 172 20 10 6](https://github.com/user-attachments/assets/af0a7e52-b853-41db-b974-9e4e727aaadc)



---

### Step 7: Victim Machine Behavior During Attack

Screenshots from the Windows 10 victim machine show what the user was experiencing while the attack was active:

- **Facebook**: The victim attempted to visit `faecbook.com` (with a typo) and received a DNS failure: "Hmmm... can't reach this page" with error `ERR_PROBE_FINISHED_NXDOMAIN`. The sniff logs on the attacker machine show repeated failed DNS lookups for facebook.com during this period.
- **Google Account**: The Google session expired ("You're not signed in") at `accounts.google.com/info/sessionExpired`. The MITM interception disrupted the session cookie verification.
- **Tesla**: The Tesla website loaded successfully, and the attacker captured the full HTTPS connection chain including CDN and Akamai edge servers.
- **Yahoo**: Yahoo loaded fully on the victim machine, and the attacker captured DNS resolution and HTTPS SNI for yahoo.com and related domains.

---

## Results and Findings

| Finding | Detail |
|---|---|
| DNS Queries Captured | gmail.com, yahoo.com, facebook.com, tesla.com, bing.com, msn.com, WhatsApp, Grammarly, Twitter/X, and more |
| HTTPS SNI Exposed | accounts.google.com, mail.google.com, tesla.com, yahoo.com, login.live.com, web.whatsapp.com |
| HTTP Responses Captured | Microsoft NCSI (ncsi.txt), partial content responses from 196.49.32.6, CDN content |
| Facebook DNS Failure | Facebook DNS repeatedly failed or returned unusual results during the attack window |
| Google Session Disrupted | MITM interference caused Google to expire the victim session |
| Windows Update Exposed | Background update download from windowsupdate.com captured without any user action |
| WhatsApp Web Visible | web.whatsapp.com HTTPS connections captured showing WhatsApp was active |
| Grammarly Extension Traffic | f-log-extension.grammarly.io connections captured, showing browser extension activity |

---

## Why HTTPS Still Leaks Information

Many people assume HTTPS makes traffic completely private. This lab shows that is not entirely true at the network level. When a browser initiates a TLS connection to a server, it sends the destination domain in the ClientHello message as the Server Name Indication (SNI) field. This field is unencrypted in standard TLS. A MITM attacker can read every domain the victim visits, even over HTTPS, even without breaking the encryption.

To fully protect against SNI leakage, the browser and server must support Encrypted Client Hello (ECH), which is not yet universally deployed.

---

## Challenges and Fixes

| Challenge | Fix Applied |
|---|---|
| Facebook DNS failures during attack | Identified as DNS resolution disruption. The DNS queries for facebook.com were captured but the responses showed failed lookups, likely due to network instability during ARP cache poisoning. Not a tool failure. |
| Google session expired on victim | Expected behavior. ARP spoofing changes the network path, which can cause session token validation to fail for security-conscious services like Google. |
| net.recon auto-starts with net.probe | Not a problem but worth noting: net.recon runs in the background automatically when net.probe is activated. This is by design in Bettercap. |

---

## Key Takeaways

- ARP has no authentication. Any device on a local network can send forged ARP replies and redirect traffic. This is a fundamental protocol weakness, not a bug in any software.
- DNS is still largely unencrypted. Every domain lookup a user performs can be read by an attacker on the same network.
- HTTPS encrypts content but not metadata. The SNI field in TLS still exposes which domains a user is visiting.
- Full duplex ARP spoofing is more reliable than one-directional spoofing because it poisons both endpoints.
- Tools like Bettercap make this attack straightforward, which is exactly why network defenders need to understand it.

---

## Security Implications and Defenses

| Defense | How It Helps |
|---|---|
| Dynamic ARP Inspection (DAI) | Validates ARP packets against a trusted database on managed switches. Blocks forged ARP replies. |
| Static ARP entries | For critical devices, setting static ARP entries prevents cache poisoning. |
| VPN usage | Encrypts all traffic between the device and VPN server, preventing a MITM on the local network from reading the content. |
| DNS over HTTPS (DoH) or DNS over TLS (DoT) | Encrypts DNS queries so domain lookups are not readable by a local attacker. |
| Encrypted Client Hello (ECH) | Hides the SNI field in TLS connections, preventing domain exposure even over HTTPS. |
| Network monitoring | Tools like Snort, Zeek, or ARP watch can detect unusual ARP activity and alert administrators. |

---

## Ethical Disclaimer

This project was conducted in a fully isolated, private lab environment using virtual machines. No real-world network, user, or system was targeted. ARP spoofing and packet sniffing on networks without explicit permission is illegal in most jurisdictions. This work is shared for educational purposes only, to help cybersecurity learners, students, and defenders understand how these attacks work so they can build better protections.

---

## Future Improvements

- Perform the same attack on an HTTPS-only site and attempt SSL stripping with `net.sniff` to compare what is and is not captured
- Configure Dynamic ARP Inspection on a managed virtual switch to demonstrate the defense in the same lab
- Use `dns.spoof` module to redirect the victim to a cloned phishing page, extending the MITM attack surface
- Capture and analyze a complete TLS handshake with Wireshark alongside Bettercap to visualize SNI leakage at the packet level
- Repeat the lab with a VPN active on the victim machine to confirm traffic is no longer readable

---

*Project completed: April 1, 2026 | Lab environment: VirtualBox, Kali Linux, Windows 10*
