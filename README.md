# DDoS Detection & Mitigation Lab

An isolated home-lab exercise simulating an **HTTP flooding (DDoS) attack**, capturing and analyzing the resulting traffic, detecting it with an IDS, and mitigating it with the Windows Firewall.

## What Is a DDoS Attack?
 
A **Distributed Denial of Service (DDoS)** attack attempts to overwhelm a network, service, or server with a flood of traffic — often from many sources at once — so it can no longer respond to legitimate users. **HTTP flooding** is a common application-layer variant: the attacker sends a large volume of seemingly valid HTTP requests to exhaust the target's connections, CPU, or bandwidth. This lab simulates that behavior in a single-attacker, controlled environment to practice detecting and responding to it, rather than launching a real multi-source distributed attack.

## Table of Contents

- [Objectives](#objectives)
- [Lab Architecture](#lab-architecture)
- [Tools Used](#tools-used)
- [1. Setting Up the Victim Web Server](#1-setting-up-the-victim-web-server)
- [2. Recon From Kali](#2-recon-from-kali)
- [3. Starting the Packet Capture](#3-starting-the-packet-capture)
- [4. Simulating the HTTP Flood](#4-simulating-the-http-flood)
- [5. Effect on the Victim](#5-effect-on-the-victim)
- [6. Traffic Analysis in Wireshark](#6-traffic-analysis-in-wireshark)
- [7. Detection With Suricata](#7-detection-with-suricata)
- [8. Mitigation With Windows Firewall](#8-mitigation-with-windows-firewall)
- [9. Verifying the Mitigation](#9-verifying-the-mitigation)
- [Results](#results)
- [Conclusion](#conclusion)

## Objectives

- Understand the principles of DDoS attacks and HTTP flooding.
- Configure an isolated virtual cybersecurity lab.
- Generate controlled HTTP traffic from Kali Linux.
- Capture traffic with `tcpdump` and analyze it with Wireshark.
- Detect excessive HTTP traffic with an IDS.
- Monitor and respond to the attack on Windows 11 using Windows Firewall.
- Evaluate the effectiveness of the mitigation.

## Lab Architecture

Three VMs on an isolated host-only virtual network:

| VM | OS | Role | IP Address |
|---|---|---|---|
| VM1 | Kali Linux | Attacker / traffic generator | `192.168.115.128` |
| VM2 | Windows 11 | Victim / web server | `192.168.115.130` |
| VM3 | Ubuntu Desktop | Monitoring / detection / analysis | `192.168.115.129` |

```
              Isolated Host-Only Network (192.168.115.0/24)
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
     Kali Linux           Windows 11          Ubuntu Desktop
  192.168.115.128        192.168.115.130        192.168.115.129
      Attacker                Victim               Monitoring
```

## Tools Used

| Tool | Purpose |
|---|---|
| Python `http.server` | Lightweight web server on the Windows victim |
| `nmap` | Service discovery from Kali |
| **DDoSify** | Controlled HTTP flood generator |
| `tcpdump` | Packet capture on the monitoring VM |
| Wireshark | Traffic analysis of the `.pcap` |
| **Suricata** | Network IDS used for detection|
| Windows Firewall / PowerShell | Mitigation |


---

## 1. Setting Up the Victim Web Server

A simple HTTP server is started on the Windows 11 ARM VM to act as the target:

```powershell
python -m http.server 8000
```

![Windows HTTP server listening on port 8000](images/01-windows-http-server.png)

## 2. Recon From Kali

An `nmap` scan from Kali confirms the web service is reachable and identifies the open port before the attack:

```bash
nmap -sV -F -T4 -n 192.168.115.130
```

`8000/tcp open http SimpleHTTPServer 0.6 (Python 3.14.7)`

![Nmap scan from Kali showing port 8000 open](images/02-kali-nmap-scan.png)

## 3. Starting the Packet Capture

On the Ubuntu monitoring VM, `tcpdump` is started on the lab interface before the traffic is generated, so the whole attack is captured:

```bash
sudo tcpdump -i enp2s0 -w DDosAttack.pcap
```

![tcpdump capturing to DDosAttack.pcap](images/03-ubuntu-tcpdump-start.png)

## 4. Simulating the HTTP Flood

DDoSify sends a controlled burst of HTTP requests from Kali to the victim:

```bash
ddosify -t http://192.168.115.130:8000 -n 1000 -d 30
```

All 1000 requests complete successfully with `HTTP 200 OK` responses, confirming the flood reached the server:

![DDoSify running against the Windows victim](images/04-ddosify-attack-running.png)
![DDoSify result — 1000/1000 success, 200 OK](images/05-ddosify-attack-result.png)

## 5. Effect on the Victim

The Windows server's access log shows the burst of `GET / HTTP/1.1` requests arriving in rapid succession from the Kali attacker:

![Windows server log flooded with GET requests](images/06-windows-server-log-flood.png)

## 6. Traffic Analysis in Wireshark

The capture is opened and inspected in Wireshark on Ubuntu.

**Full capture** — 11,634 packets recorded, including the ARP/ICMP connectivity checks and the TCP handshakes preceding the flood:

![Wireshark opened on DDOS_Attack.pcap](images/07-wireshark-pcap-overview.png)

**Filtered on `http`** — isolates the repeated `GET / HTTP/1.1` requests and `200 OK` responses between the attacker and the victim:

![Wireshark filtered by http protocol](images/08-wireshark-http-filter.png)

**Filtered on `tcp.port == 8000`** — shows the SYN/SYN-ACK/ACK handshakes stacking up alongside the HTTP requests, characteristic of a flood rather than isolated traffic:

![Wireshark filtered by tcp.port == 8000](images/09-wireshark-port-filter.png)

## 7. Detection With Suricata

**Suricata** is an open-source Network Intrusion Detection and Prevention System (IDS/IPS). It inspects live traffic against a set of signatures/rules and can log matches, raise alerts, or drop traffic outright depending on the mode it's run in. Here it's used in IDS mode — reading traffic on the monitoring interface and generating an alert whenever a source exceeds the request threshold defined in the custom rule below.

A custom local rule is added to detect a high rate of SYN/connection attempts from a single source toward the victim's web port:

```
alert tcp 10.0.0.11 any -> 10.0.0.12 8000 (msg:"LAB HTTP Flooding Detected"; flags:S; threshold:type threshold, track by_src, count 100, seconds 10; sid:1000001; rev:1;)
```

![Suricata rule in local.rules (part 1)](images/10-suricata-rule-part1.png)
![Suricata rule in local.rules (part 2)](images/11-suricata-rule-part2.png)

Suricata is then launched in IDS mode on the lab interface:

```bash
sudo suricata -i br0 -c /etc/suricata/suricata.yaml
```

![Suricata engine started](images/12-suricata-started.png)

The attack is re-run to trigger the rule:

![Second DDoSify run to trigger detection](images/13-second-attack-run.png)
![Attack run result](images/14-attack-run-failed-result.png)

**Alert fired** — Suricata flags the flood as an attempted Denial of Service from the Kali attacker to the Windows victim on port 8000:

```
[**] [1:1000001:1] LAB HTTP Flooding Detected [**]
[Classification: Attempted Denial of Service] [Priority: 1] {TCP} 192.168.115.128:54321 -> 192.168.115.130:8000
```

![Suricata alert: LAB HTTP Flooding Detected](images/15-suricata-alert-detected.png)

## 8. Mitigation With Windows Firewall

Once the attacker's IP is identified from the Suricata alert, an inbound block rule is created on the Windows victim via PowerShell:

```powershell
New-NetFirewallRule `
  -DisplayName "LAB - Block DDoS Attacker" `
  -Direction Inbound `
  -Action Block `
  -RemoteAddress 192.168.115.128 `
  -Profile Any
```

![Firewall rule created](images/16-firewall-rule-created.png)

The rule is then verified:

```powershell
Get-NetFirewallRule -DisplayName "LAB - Block DDoS Attacker" |
  Format-List DisplayName,Enabled,Direction,Action,Profile
```

`Enabled: True · Direction: Inbound · Action: Block · Profile: Any`

![Firewall rule verified as enabled](images/17-firewall-rule-verified.png)

## 9. Verifying the Mitigation

With the rule active, requests from the Kali attacker no longer reach the web server:

```bash
curl --connect-timeout 5 http://192.168.115.130:8000
# curl: (28) Connection timed out after 5004 milliseconds
```

A repeat of the DDoSify flood also fails to reach the server once the rule is in place, confirming the mitigation holds under sustained traffic, not just a single request.

![curl request timing out after the firewall rule is applied](images/18-curl-blocked-after-mitigation.png)

A repeat of the full DDoSify flood also fails to reach the server once the rule is in place, confirming the mitigation holds under sustained traffic, not just a single request. All 1000 requests fail with `connection timeout`:
 
```
Success Count:   0     (0%)
Failed Count:    1000  (100%)
Server Error Distribution (Count:Reason): 1000 : connection timeout
Test Status : Success
```
 
![DDoSify flood failing against the victim after the firewall rule is applied](images/19-ddos-blocked-after-mitigation.png)

## Results

| Indicator | Before Mitigation | After Mitigation |
|---|---|---|
| HTTP response | `200 OK`, server reachable | Connection times out |
| DDoSify test status | 1000/1000 successful | 0/1000 successful (connection refused / timed out) |
| Suricata alert | `LAB HTTP Flooding Detected` fired | Traffic blocked before reaching the server |
| Windows Firewall rule | Not present | `LAB - Block DDoS Attacker` — Enabled, Inbound, Block |
| Web server availability (to attacker) | Available | Unavailable |


## Conclusion

This lab demonstrated the full lifecycle of a DDoS-style incident in a controlled environment: generating a controlled HTTP flood, capturing and analyzing the traffic in Wireshark, detecting the flood with a threshold-based Suricata rule, and mitigating it with a Windows Firewall rule blocking the attacker's IP. The before/after comparison confirms the firewall rule was effective against a single-source flood, while also highlighting why IP-based blocking alone doesn't scale to attacks distributed across many sources.
