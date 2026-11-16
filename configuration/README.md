# Suricata Configuration

## local.rules

The lab signatures are in [local.rules](local.rules). Copy them to the rule directory configured by `default-rule-path` in `/etc/suricata/suricata.yaml`.

Example lab path:

```bash
sudo install -D -m 0644 local.rules /var/lib/suricata/rules/local.rules
```

Ensure the main Suricata configuration loads the file:

```yaml
default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
  - local.rules
```

The AF_PACKET interface used in the documented lab was `ens37`. Interface names vary; inspect `ip -br addr` and select the interface that actually receives the lab traffic.

Validate after editing:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

## Lab-specific assumptions

- Kali source: `192.168.183.128`
- Ubuntu destination: `192.168.183.129`
- Test service: TCP/8080
- Capture interface: `ens37`

SID 1000004 is pinned to these lab IP addresses. Update the rule for your own isolated network. The HTTP drop rule SID 1000003 was configured but its successful blocking was not conclusively verified in the recorded test.
