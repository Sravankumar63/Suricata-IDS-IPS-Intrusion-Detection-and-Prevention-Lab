# Suricata IDS/IPS — Network Intrusion Detection and Prevention Lab

A hands-on network security lab using **Suricata 8.0.8**, **Ubuntu Server**, **Kali Linux**, **iptables**, and **NFQUEUE**. It demonstrates custom HTTP and TCP SYN detection, alert investigation, and a controlled TCP packet-drop test.

> **Scope:** This is an academic lab built in VMware Workstation. It is intended for controlled learning and defensive testing, not production deployment.

## Lab architecture

| Component | Role | Lab address |
|---|---|---|
| Ubuntu Server | Suricata sensor / IPS test host and temporary HTTP server | `192.168.183.129` |
| Kali Linux | Traffic generation and testing | `192.168.183.128` |
| VMware Workstation | Virtualization platform | Isolated lab network |

- Suricata version: **8.0.8**
- Ubuntu capture interface: **ens37**
- Temporary HTTP service: **TCP/8080**
- Alert logs: `/var/log/suricata/fast.log` and `/var/log/suricata/eve.json`

## Objectives

- Configure Suricata to monitor traffic on the correct Ubuntu interface.
- Create custom signatures for HTTP URI detection and TCP SYN scan detection.
- Generate controlled HTTP and TCP scan traffic from Kali Linux.
- Investigate alerts using `fast.log` and `eve.json`.
- Integrate Suricata with iptables NFQUEUE and demonstrate a controlled TCP drop.

## Custom rules

The lab uses four local signatures. SID 1000004 contains lab-specific IP addresses; change them before reusing the rule in another environment.

| SID | Action | Purpose | Recorded result |
|---|---|---|---|
| 1000001 | alert | Detect HTTP requests containing `/admin` on TCP/8080 | Detection observed |
| 1000002 | alert | Detect repeated TCP SYN packets using a threshold | Detection observed |
| 1000003 | drop | Attempt to prevent HTTP `/admin` requests | Configured; successful blocking not conclusively verified |
| 1000004 | drop | Drop matching TCP SYN traffic from Kali to Ubuntu TCP/8080 | Blocked event observed in `eve.json` and `fast.log` |

### `configuration/local.rules`

The four rules are provided in [configuration/local.rules](configuration/local.rules).

## Configuration notes

The system configuration was edited at:

```bash
sudo nano /etc/suricata/suricata.yaml
```

Relevant settings used in the lab:

- AF_PACKET interface: `ens37`
- Default rule path: `/var/lib/suricata/rules`
- Rule files: `suricata.rules` and `local.rules`

The repository contains the lab's local rule file. It does not include a complete machine-specific `suricata.yaml`; apply the relevant settings to the configuration installed on your Ubuntu VM.

## Implementation and testing

### 1. Verify VM connectivity

On Ubuntu Server:

```bash
ip -br addr
ip route
ip route get 192.168.183.128
ping -c 4 192.168.183.128
```

On Kali Linux:

```bash
ip -br addr
ping -c 4 192.168.183.129
```

The recorded lab addresses were Ubuntu `192.168.183.129` and Kali `192.168.183.128`. The selected Ubuntu capture interface was `ens37`.

### 2. Install and inspect Suricata

```bash
sudo apt update
sudo apt install suricata jq -y
sudo suricata --build-info
sudo systemctl status suricata
```

The lab used Suricata 8.0.8. An initial service startup failure was traced to an unavailable `eth0` interface reference; the AF_PACKET interface was corrected to `ens37`.

### 3. Configure local rules and validate

Confirm the rule path and `rule-files` section in `/etc/suricata/suricata.yaml`, then place the custom signatures in the configured local rule file.

```bash
sudo mkdir -p /var/lib/suricata/rules
sudo nano /var/lib/suricata/rules/local.rules
sudo suricata -T -c /etc/suricata/suricata.yaml
sudo systemctl restart suricata
sudo systemctl status suricata
```

The recorded `suricata-update` run loaded distribution-provided rules but could not download the Emerging Threats archive because DNS resolution failed. That feed download should not be described as successful.

### 4. Start a temporary HTTP server

On Ubuntu Server:

```bash
mkdir -p ~/ids-lab
cd ~/ids-lab
nohup python3 -m http.server 8080 > ~/ids-lab/server.log 2>&1 &
ss -lntp | grep 8080
```

From Kali Linux:

```bash
curl -i http://192.168.183.129:8080/
curl -i http://192.168.183.129:8080/admin
```

The root request returned HTTP 200. The `/admin` request returned HTTP 404 because that resource was not present in the Python server's document root. Suricata separately generated the SID 1000001 URI detection alert. The HTTP 404 response is **not** evidence of IPS blocking.

### 5. Generate a TCP SYN scan

From Kali Linux, against the authorized lab VM only:

```bash
sudo nmap -sS -T4 -p 1-1000,8080 192.168.183.129
```

Inspect Suricata alerts on Ubuntu:

```bash
sudo tail -f /var/log/suricata/fast.log
```

The custom SYN scan signature was observed as SID 1000002. Ensure screenshots and report text only claim that port 8080 was open when the relevant Nmap output explicitly shows `8080/tcp open`.

### 6. Investigate alerts

```bash
sudo jq 'select(.event_type=="alert")' /var/log/suricata/eve.json
```

For selected fields:

```bash
sudo jq 'select(.event_type=="alert") | {timestamp,src_ip,dest_ip,alert}' /var/log/suricata/eve.json
```

Review the timestamp, source and destination addresses, signature ID, signature message, and alert action. An `allowed` action on an alert-only signature means the traffic was detected and logged, not blocked.

### 7. Configure NFQUEUE for the controlled IPS test

The NFQUEUE kernel module was checked and loaded:

```bash
sudo modprobe nfnetlink_queue
lsmod | grep nfnetlink_queue
```

The documented inbound queue rule was:

```bash
sudo iptables -I INPUT 1 -i ens37 -s 192.168.183.128 -p tcp --dport 8080 -j NFQUEUE --queue-num 0 --queue-bypass
sudo iptables -vnL INPUT --line-numbers
```

The systemd-managed Suricata service was stopped before starting a separate queue-mode process:

```bash
sudo systemctl stop suricata
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

While Suricata remained running in queue mode, the controlled connection was generated from Kali:

```bash
curl --connect-timeout 5 -v http://192.168.183.129:8080/
```

Verify SID 1000004 on Ubuntu:

```bash
sudo tail -n 20 /var/log/suricata/fast.log
sudo jq 'select(.event_type=="alert" and .alert.signature_id==1000004)' /var/log/suricata/eve.json
```

The recorded test showed SID 1000004 with `alert.action: blocked` in `eve.json` and corresponding `[Drop]` entries in `fast.log`.

> **Configuration caveat:** NFQUEUE rules and inline traffic direction must be designed consistently for both packet directions in a real deployment. This repository describes the observed lab test, not a hardened production firewall policy. Review and remove temporary iptables rules before reusing the VM.

## Results and limitations

### Demonstrated

- Suricata loaded its configuration and ran on the selected Ubuntu interface.
- SID 1000001 generated an HTTP `/admin` detection alert.
- SID 1000002 generated TCP SYN scan detection alerts.
- SID 1000004 produced a verified blocked event in `eve.json` and drop entries in `fast.log`.

### Not conclusively verified

- SID 1000003 was configured as an HTTP drop rule, but the available test evidence did not conclusively show a blocked event for that SID.
- The Emerging Threats rule download failed during the recorded update attempt because DNS resolution failed.

## Repository layout

```text
.
├── README.md
├── configuration/
│   └── local.rules
├── documentation/
│   └── README.md
├── screenshots/
│   └── README.md
└── testing/
    └── test-commands.md
```

The final project report and reviewed screenshots can be added under `documentation/` and `screenshots/`. Check images for personal information, passwords, tokens, or unrelated VM details before publishing.

## Safety

- Scan only systems you own or are explicitly authorized to test.
- Keep this lab isolated from production networks.
- The rules contain lab-specific IP addresses and port assumptions.
- Review firewall direction, queue bypass behaviour, and cleanup before adapting the example for another environment.

## References

- [Suricata Documentation](https://docs.suricata.io/)
- [Suricata configuration guide](https://docs.suricata.io/en/latest/configuration/suricata-yaml.html)
- [Suricata IPS / inline mode](https://docs.suricata.io/en/latest/ips/ips-mode.html)
- [Suricata rule management](https://docs.suricata.io/en/latest/rule-management/index.html)
- [Nmap Reference Guide](https://nmap.org/book/man.html)

---

**Project type:** Academic hands-on cybersecurity lab  
**Environment:** VMware Workstation, Ubuntu Server, Kali Linux  
**Focus:** Network IDS, custom signatures, alert analysis, and NFQUEUE-based IPS testing
