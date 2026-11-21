# Project 15 — MAC Changer and Wazuh Open-Source SIEM

**Author:** Vishnu Vardhan  
**Wazuh version:** 4.14.10  
**Environment:** Oracle VM VirtualBox · Kali Linux · Ubuntu Server / Ubuntu Target

## Project overview
This project documents two hands-on cybersecurity exercises in an isolated virtual lab:

1. **MAC address manipulation with Macchanger** — inspect the interface, apply random and custom MAC addresses, restore the permanent address, and verify network connectivity and ARP/neighbour information.
2. **Endpoint monitoring with Wazuh SIEM** — deploy Wazuh components, enroll an Ubuntu endpoint, enable File Integrity Monitoring (FIM), modify a monitored test file, and verify the resulting syscheck alert.

## Table of contents
- [Lab architecture](#lab-architecture)
- [Tools and components](#tools-and-components)
- [Part A — MAC Changer](#part-a--mac-changer)
- [Part B — Wazuh deployment](#part-b--wazuh-deployment)
- [Wazuh Agent enrollment](#wazuh-agent-enrollment)
- [File Integrity Monitoring](#file-integrity-monitoring)
- [Results](#results)
- [Repository structure](#repository-structure)
- [Safety and limitations](#safety-and-limitations)

## Lab architecture

| VM / component | Role | Lab address |
|---|---|---|
| Kali Linux | MAC changer and network testing | `192.168.56.103/24` · interface `eth1` |
| Ubuntu Target | Monitored endpoint | `192.168.56.107` |
| Wazuh Server | Manager, Indexer, Dashboard | `192.168.56.109` |
| Wazuh Agent | Endpoint telemetry | Agent `001` · `ubuntu-server` · Active |

VirtualBox used NAT for internet access and a host-only adapter for VM-to-VM communication. The endpoint connected to the Manager at `192.168.56.109` over TCP port `1514`. Dashboard URL: `https://192.168.56.109`.

## Tools and components
- Kali Linux and Macchanger
- Ubuntu Server / Ubuntu Target
- Wazuh 4.14.10: Indexer, Manager, Dashboard and Agent
- Filebeat
- Oracle VM VirtualBox
- Wireshark (ARP capture review)

## Part A — MAC Changer

Identify the actual interface first. `eth1` is the interface recorded in this lab; interface names can differ between systems.

```bash
ip link show
ip addr show
macchanger -s eth1
```

Randomized address:
```bash
sudo ip link set eth1 down
sudo macchanger -r eth1
sudo ip link set eth1 up
macchanger -s eth1
```

Custom locally administered address:
```bash
sudo ip link set eth1 down
sudo macchanger -m 02:11:22:33:44:55 eth1
sudo ip link set eth1 up
macchanger -s eth1
```

Restore the permanent address:
```bash
sudo ip link set eth1 down
sudo macchanger -p eth1
sudo ip link set eth1 up
macchanger -s eth1
```

Connectivity and neighbour checks:
```bash
ping -c 4 192.168.56.107
ip neigh
arp -n
```

**Recorded values:** permanent `08:00:27:25:6f:10`; randomized example `aa:44:34:83:9d:81`; custom example `02:11:22:33:44:55`. After restoration, ping returned 4/4 replies and 0% packet loss. The ARP capture is in `evidence/macchanger/project15_arp_capture.pcapng`.

## Part B — Wazuh deployment

Wazuh `4.14.10` was installed on the Ubuntu Wazuh Server VM. The recorded deployment workflow included restoring and validating Indexer certificates/permissions, completing Indexer security initialization, starting the Indexer, installing/starting the Manager and Dashboard, and starting Filebeat.

The original package-install command transcript was not present in the supplied evidence. Therefore this repository documents the observed deployment workflow and verified service checks, not an invented installer command sequence.

Verify the services on the Wazuh Server:
```bash
sudo systemctl is-active wazuh-indexer
sudo systemctl is-active wazuh-manager
sudo systemctl is-active wazuh-dashboard
sudo systemctl is-active filebeat
sudo /var/ossec/bin/agent_control -l
```

The recorded checks showed the services active and agent `001` (`ubuntu-server`) Active.

## Wazuh Agent enrollment

Wazuh Agent `4.14.10` was installed on the Ubuntu Target and configured to connect to Manager `192.168.56.109` over TCP port `1514`.

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent --no-pager
sudo grep -i "connected to the server" /var/ossec/logs/ossec.log | tail
```

The endpoint log recorded a successful connection to `192.168.56.109:1514/tcp`. Dashboard Endpoints showed agent `001`, `ubuntu-server`, `192.168.56.107`, version `4.14.10`, status `Active`.

## File Integrity Monitoring

The endpoint's `/var/ossec/etc/ossec.conf` contained this `syscheck` configuration:

```xml
<syscheck>
  <disabled>no</disabled>
  <frequency>43200</frequency>
  <scan_on_start>yes</scan_on_start>
  <directories realtime="yes">/etc</directories>
  <directories>/usr/bin,/usr/sbin</directories>
  <directories>/bin,/sbin,/boot</directories>
</syscheck>
```

The agent was restarted after the configuration change:
```bash
sudo systemctl restart wazuh-agent
sudo systemctl is-active wazuh-agent
```

A controlled test file was created and modified on the Ubuntu Target:
```bash
sudo touch /etc/project15-wazuh-test.txt
echo "Project 15 FIM verification" | sudo tee -a /etc/project15-wazuh-test.txt
```

On the Wazuh Server, the alert was checked with:
```bash
sudo grep 'project15-wazuh-test.txt' /var/ossec/logs/alerts/alerts.json | tail -5
```

The supplied alert evidence recorded a syscheck integrity-change event for `/etc/project15-wazuh-test.txt`, agent `001`, rule ID `550`, level `7`, description `Integrity checksum changed`.

## Results

| Test | Recorded result |
|---|---|
| MAC random/custom/restore | Completed; permanent address restored |
| Connectivity | 4/4 replies; 0% packet loss |
| ARP/neighbour inspection | Completed; PCAPNG retained |
| Wazuh services | Indexer, Manager, Dashboard, Filebeat active |
| Agent enrollment | Agent `001` `ubuntu-server` Active |
| Agent connection | `192.168.56.109:1514/TCP` |
| FIM test | syscheck file-modified event recorded |

## Repository structure

```text
Project15_Final_Package/
├── README.md
├── .gitignore
├── docs/
│   ├── Project15_Final_Technical_Report.pdf
│   └── Project15_Final_Technical_Report.docx
├── screenshots/
│   ├── wazuh-indexer-active.png
│   ├── wazuh-agent-active.png
│   ├── fim-configuration.png
│   ├── wazuh-fim-alert.png
│   └── README.md
├── configs/
│   └── fim-syscheck-example.xml
└── evidence/
    └── macchanger/
        ├── *.txt
        └── project15_arp_capture.pcapng
```

## Safety and limitations
- This is an authorized educational lab, not a production security audit or formal vulnerability assessment.
- A MAC change does not guarantee anonymity or bypass network controls.
- FIM detects changes to configured monitored paths; it does not automatically block, quarantine or remediate them.
- Never commit passwords, tokens, private keys, certificates, VM disks or exported VM appliances.
- Review screenshots and packet captures before making the repository public.
- Separate original MAC changer screenshots were not included in the uploaded image files; add authentic screenshots if you have them. Do not fabricate evidence.

## Author
Vishnu Vardhan
