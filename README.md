## Obiettivo del laboratorio
Implementazione pratica di OSSEC HIDS su VM Azure per:

File Integrity Monitoring (FIM) su sistema critico

Rilevamento brute force SSH (Rule 5710)

Active Response automatico (iptables DROP)

Threat hunting su attacchi reali

## Architettura

Azure (West Europe)
├── RG: RG-CyberLab  
├── VM: Ubuntu 22.04 LTS (B1s, IP pubblico: VM-Suricata)
│   ├── OSSEC HIDS v3.7.0 (standalone server)
│   ├── Syscheck realtime (/etc, /usr/bin)
│   └── Active Response (level 5)
└── NSG: SSH esposto (simula ambiente reale)

## Guida riproducibile

1) Azure VM deployment
Portale Azure → Virtual Machines → Ubuntu 22.04 → B1s → SSH key → NSG SSH pubblico.

2) Accesso e preparazione

ssh azureuser@IP_VM
sudo apt update && sudo apt upgrade -y

3) Installazione OSSEC

wget -q -O - https://updates.atomicorp.com/installers/atomic | sudo bash
sudo apt update
sudo apt install ossec-hids-server -y

4) Configurazione Ubuntu-optimized

sudo nano /var/ossec/etc/ossec.conf

<!-- Solo log Ubuntu -->
<localfile><log_format>syslog</log_format><location>/var/log/auth.log</location></localfile>
<localfile><log_format>syslog</log_format><location>/var/log/syslog</location></localfile>

<!-- Active Response aggressivo -->
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <level>5</level>
  <timeout>1800</timeout>
</active-response>

5) Hardening specifico

# DNS chroot fix
sudo cp /etc/resolv.conf /var/ossec/etc/
sudo cp /etc/hosts /var/ossec/etc/

# Queue permissions
sudo chown -R ossec:ossec /var/ossec/queue
sudo chmod -R 770 /var/ossec/queue

sudo /var/ossec/bin/ossec-control restart

6) SOC Dashboard live

watch -n 2 "tail -3 /var/ossec/logs/alerts/alerts.log && iptables -L | grep DROP"

Risultati tipici (dal mio lab)
Attacchi reali rilevati

209.38.102.112: 40+ tentativi (test, ubuntu, guest, oracle)
134.199.194.122: postgres brute force
194.195.xxx.xxx: multiple users

Rule 5710 (level 5): Invalid user login
Status: All IPs BLOCKED (Active Response)

Test FIM Syscheck

echo "TEST" >> /etc/motd

Rule 550 (level 7): Integrity checksum changed for '/etc/motd'
MD5: a561687... → a60f88a... (Size: 13→26)

Active Response Log

Active response 'firewall-drop' executed for 209.38.102.112
iptables: DROP 209.38.102.112 (1800s timeout)

Skills dimostrate
OSSEC HIDS deployment e troubleshooting (repository, chroot DNS)

Ubuntu-optimized configuration (no logcollector errors)

Active Response implementation (level 5 brute force blocking)

Live threat hunting (3 botnet IPs documented)

SOC dashboard scripting (alerts + iptables correlation)

