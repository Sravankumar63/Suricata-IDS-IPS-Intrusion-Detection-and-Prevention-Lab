# Test Commands

These commands were used in the isolated VMware lab. Run them only against systems you own or are authorized to test.

## Connectivity

Ubuntu:
```bash
ip -br addr
ip route
ping -c 4 192.168.183.128
```

Kali:
```bash
ip -br addr
ping -c 4 192.168.183.129
```

## HTTP service

Ubuntu:
```bash
mkdir -p ~/ids-lab
cd ~/ids-lab
nohup python3 -m http.server 8080 > ~/ids-lab/server.log 2>&1 &
ss -lntp | grep 8080
```

Kali:
```bash
curl -i http://192.168.183.129:8080/
curl -i http://192.168.183.129:8080/admin
```

## Nmap TCP SYN scan

Kali:
```bash
sudo nmap -sS -T4 -p 1-1000,8080 192.168.183.129
```

## Suricata logs

Ubuntu:
```bash
sudo tail -n 30 /var/log/suricata/fast.log
sudo jq 'select(.event_type=="alert")' /var/log/suricata/eve.json
```

## NFQUEUE test

Ubuntu:
```bash
sudo modprobe nfnetlink_queue
sudo iptables -I INPUT 1 -i ens37 -s 192.168.183.128 -p tcp --dport 8080 -j NFQUEUE --queue-num 0 --queue-bypass
sudo systemctl stop suricata
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

In Kali, while Suricata runs in queue mode:
```bash
curl --connect-timeout 5 -v http://192.168.183.129:8080/
```

Inspect SID 1000004 on Ubuntu:
```bash
sudo jq 'select(.event_type=="alert" and .alert.signature_id==1000004)' /var/log/suricata/eve.json
```

## Expected evidence

- SID 1000001: HTTP `/admin` detection.
- SID 1000002: TCP SYN scan detection.
- SID 1000003: HTTP prevention rule configured; blocking was not conclusively verified in this lab.
- SID 1000004: controlled TCP SYN drop event recorded as blocked.

The HTTP 404 response from Python's server is not evidence of Suricata prevention. It means the requested `/admin` resource was not present in the server's document root.
