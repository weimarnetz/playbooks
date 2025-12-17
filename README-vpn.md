# VPN-Server Playbook für Weimarer Freifunk

Dieses Ansible-Playbook automatisiert die Einrichtung von VPN-Servern für die Weimarer Freifunk-Community. Es konfiguriert WireGuard für die Server-zu-Server Verbindungen, OLSR für das Routing, fastd für Router-Verbindungen, iptables-Regeln für NAT-Masquerading und einen Webserver für Status-Informationen.

## Architektur

Die VPN-Server-Architektur besteht aus:

- **WireGuard**: Erstellt verschlüsselte Verbindungen zwischen den VPN-Servern (inter-VPN-Verbindung)
- **OLSR**: Optimized Link State Routing Protocol für das Mesh-Routing zwischen den Servern über WireGuard und fastd
- **fastd**: VPN-Verbindungen für Router zum Server
- **iptables**: NAT-Masquerading für Pakete außerhalb 10.0.0.0/8 und Rate-Limiting
- **Nginx**: Webserver für statisches JSON-Status-Endpoint
- **Netzwerk**: Alle Router verwenden das 10.0.0.0/8 Netzwerk

## Voraussetzungen

- Ansible installiert (Version 2.9 oder höher)
- Zugriff auf die VPN-Server (SSH)
- sudo/root-Rechte auf den Servern
- Internet-Verbindung für Downloads (OLSR-Quellcode, Packages)

## Schnellstart

### 1. Inventory-Datei erstellen

Erstelle eine Inventory-Datei (`inventory-vpn.yml`):

**Wichtige Ansible-Variablen:**
- `ansible_host`: IP-Adresse oder Hostname (nur wenn Inventory-Name != Verbindungsadresse)
- `ansible_port`: SSH-Port (Standard: 22, nur wenn abweichend)
- `ansible_user`: SSH-User (Standard: aktueller User, nur wenn abweichend)

**Beispiel Inventory:**

```yaml
---
# inventory-vpn.yml
vpn_servers:
  hosts:
    5.v.weimarnetz.de:
      vpn_server_number: 5
      ansible_port: <port>
      ansible_user: <username>
      # Weitere Variablen siehe Abschnitt "Variablen"
```

**Automatische Berechnungen basierend auf `vpn_server_number`:**

- **WireGuard IP-Adresse**: Server 2 → `10.63.1.5/30`, Server 3 → `10.63.1.9/30`, etc.
  - Formel: `10.63.1.((vpn_server_number - 1) * 4 + 1)/30`
- **WireGuard Port**: Server 2 → `51192`, Server 3 → `51193`, etc.
  - Formel: `wireguard_base_port + vpn_server_number` (Standard: `51190`)
- **OLSR MainIP**: Wird automatisch aus WireGuard-Adresse gesetzt
- **fastd Gateway-IP**: `10.63.0.<vpn_server_number>` (z.B. Server 2 → `10.63.0.2`)
- **VPN-Status Gateway-IP**: `10.63.0.<vpn_server_number>` (z.B. Server 2 → `10.63.0.2`)

### 2. Sensible Variablen setzen

Die folgenden Variablen **MÜSSEN** gesetzt werden (entweder im Inventory oder per Command-Line mit `-e`):

- `wireguard_peer_public_key`: Öffentlicher Schlüssel des WireGuard-Peers (zentraler Server)
- `wireguard_peer_endpoint_host`: Public IP-Adresse oder Hostname des Peers (Standard: `77.87.48.19`)
- `fastd_secret`: fastd Secret (privater Schlüssel)

**Option 1: Im Inventory verschlüsselt mit Ansible Vault (empfohlen):**

```yaml
# inventory-vpn.yml
vpn_servers:
  hosts:
    5.v.weimarnetz.de:
      vpn_server_number: 5
      wireguard_peer_public_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          663864396539663162646264663732663736656638633362643761666562653962...
      fastd_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          366336643766373831316264346663653765666638633362643761666562653962...
```

**Variablen verschlüsseln:**

```bash
ansible-vault encrypt_string 'PEER_PUBLIC_KEY_HIER' --name 'wireguard_peer_public_key'
ansible-vault encrypt_string 'fastd_secret_hier' --name 'fastd_secret'
```

**Option 2: In `group_vars/vpn_servers/vault.yml` (empfohlen für mehrere Server):**

```yaml
# group_vars/vpn_servers/vault.yml
---
wireguard_peer_public_key: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          663864396539663162646264663732663736656638633362643761666562653962...
fastd_secret: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          366336643766373831316264346663653765666638633362643761666562653962...
```

**Option 3: Im Inventory unverschlüsselt (nicht empfohlen für Produktion):**

```yaml
# inventory-vpn.yml
vpn_servers:
  hosts:
    5.v.weimarnetz.de:
      vpn_server_number: 5
      wireguard_peer_public_key: "PEER_PUBLIC_KEY_HIER"
      fastd_secret: "fastd_secret_hier"
```

**Option 4: Per Command-Line (nicht empfohlen für Produktion):**

```bash
ansible-playbook -i inventory-vpn.yml vpn-server-playbook.yml \
  -e "wireguard_peer_public_key=PEER_KEY" \
  -e "fastd_secret=SECRET" \
  --ask-vault-pass
```

### 3. Playbook ausführen

```bash
# Mit verschlüsselten Variablen
ansible-playbook -i inventory-vpn.yml vpn-server-playbook.yml --ask-vault-pass

# Mit verschlüsselten Variablen und sudo-Passwort-Abfrage (falls auf dem Zielsystem für sudo ein Passwort benötigt wird)
ansible-playbook -i inventory-vpn.yml vpn-server-playbook.yml --ask-vault-pass --ask-become-pass

# Ohne verschlüsselte Variablen
ansible-playbook -i inventory-vpn.yml vpn-server-playbook.yml
```

**Hinweis:** Wenn Variablen nicht gesetzt sind, schlägt das Playbook mit einer aussagekräftigen Fehlermeldung fehl.

### 4. Outputs

Nach dem erfolgreichen Ausführen des Playbooks werden wichtige Informationen ausgegeben:

**WireGuard öffentlicher Schlüssel:**
Der öffentliche WireGuard-Schlüssel wird am Ende des Playbook-Laufs für jeden Server ausgegeben. Dieser Schlüssel muss an die Admins des zentralen Servers weitergegeben werden, damit der neue VPN-Server als Peer konfiguriert werden kann.

Beispiel-Output:
```
==========================================
WireGuard Setup abgeschlossen
==========================================
Server: 5.v.weimarnetz.de (vpn_server_number: 5)

ÖFFENTLICHER SCHLÜSSEL (für zentralen Server):
abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ123456==

WireGuard Konfiguration:
  Interface: wg0
  Adresse: 10.63.1.17/30
  Port: 51195
  Endpoint: 77.87.48.19:51195

Bitte übermittle den öffentlichen Schlüssel an die
Admins des zentralen Servers, damit dieser konfiguriert werden kann.
==========================================
```

**Hinweis:** Die Outputs werden immer angezeigt (auch bei wiederholten Playbook-Läufen ohne Änderungen), sodass die Schlüssel jederzeit verfügbar sind.

## Variablen

### Erforderliche Variablen

| Variable | Beschreibung | Setzen in |
|----------|-------------|-----------|
| `vpn_server_number` | Server-Nummer (2-9) | Inventory (pro Server) |
| `wireguard_peer_public_key` | Öffentlicher Schlüssel des WireGuard-Peers | Inventory oder `-e` |
| `fastd_secret` | fastd Secret (privater Schlüssel) | Inventory oder `-e` |

### WireGuard-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `wireguard_interface` | Name des WireGuard-Interfaces | `wg0` |
| `wireguard_base_port` | Basis-Port (Server-Nummer wird hinzugefügt) | `51190` |
| `wireguard_ip_base` | Basis-IP für /30-Netze | `10.63.1` |
| `wireguard_peer_endpoint_host` | Public IP-Adresse oder Hostname des Peers | `77.87.48.19` |
| `wireguard_peer_allowed_ips` | AllowedIPs für Peer (optional) | Auto-generiert |
| `wireguard_persistent_keepalive` | Keep-Alive Intervall | `20` |

### iptables-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `iptables_udp698_rate_limit` | Rate-Limit für UDP Port 698 | `200/sec` |
| `iptables_udp698_burst` | Burst für UDP Port 698 Rate-Limiting | `200` |
| `iptables_masquerade_networks` | Netzwerke für NAT-Masquerading | `["10.63.0.0/16", "10.64.0.0/16"]` |
| `iptables_exclude_networks` | Netzwerke die nicht maskiert werden | `["10.0.0.0/8"]` |

### fastd-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `fastd_port` | Port für fastd | `10000` |
| `fastd_interface` | Interface-Name | `fastd_mesh` |
| `fastd_user` | User für fastd-Daemon | `nobody` |
| `fastd_mtu` | MTU für fastd | `1280` |
| `fastd_log_level` | Log-Level | `debug` |
| `fastd_gateway_ip_base` | Basis-IP für Gateway | `10.63.0` |

### OLSR-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `olsr_branch` | Git-Branch des OLSR-Quellcodes | `master` |
| `olsr_main_ip` | Main IP-Adresse des Servers | Auto (aus WireGuard) |
| `olsr_wireguard_interface` | WireGuard-Interface für OLSR | `wg0` |
| `olsr_fastd_interface` | fastd-Interface für OLSR (optional) | `fastd_mesh` |
| `olsr_hna4_networks` | Liste von HNA4-Netzwerken | `[]` |
| `olsr_plugins` | Plugin-Konfigurationen | siehe defaults |

### VPN-Status-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `vpn_status_gateway_ip_base` | Basis-IP für Gateway | `10.63.0` |
| `vpn_status_country` | Länder-Code für JSON | `DE` |
| `vpn_status_maxmtu` | MTU für JSON | `1280` |
| `vpn_status_clients` | Client-Anzahl für JSON | `0` |

## Generierte Dateien

Nach dem Ausführen des Playbooks werden folgende Dateien erstellt:

**WireGuard:**
- `/etc/wireguard/wg0_private.key` - Privater Schlüssel (nicht weitergeben!)
- `/etc/wireguard/wg0_public.key` - Öffentlicher Schlüssel (für Peer-Konfiguration)
- `/etc/wireguard/wg0.conf` - WireGuard-Konfiguration

**fastd:**
- `/etc/fastd/vpn/fastd.conf` - fastd-Konfiguration
- `/etc/fastd/vpn/peers/` - Verzeichnis für Peer-Konfigurationen

**iptables:**
- `/etc/iptables/rules.v4` - iptables-Regeln

**OLSR:**
- `/etc/olsrd/olsrd.conf` - OLSR-Konfiguration
- `/etc/systemd/system/olsrd.service` - Systemd-Service-Datei
- `/usr/local/bin/neigh.sh` - Script zur Anzeige von OLSR-Nachbarn

**VPN-Status:**
- `/var/www/html/freifunk/vpn/index.json` - Statische JSON-Datei mit Server-Status

## Troubleshooting

### WireGuard-Verbindung funktioniert nicht

1. Prüfe, ob der Service läuft:
   ```bash
   systemctl status wg-quick@wg0
   ```

2. Prüfe die WireGuard-Konfiguration:
   ```bash
   wg show
   ```

3. Prüfe die Logs:
   ```bash
   journalctl -u wg-quick@wg0 -f
   ```

4. Stelle sicher, dass die öffentlichen Schlüssel auf beiden Seiten korrekt sind

### OLSR findet keine Peers

1. Prüfe, ob der Service läuft:
   ```bash
   systemctl status olsrd
   ```

2. Prüfe die OLSR-Konfiguration:
   ```bash
   cat /etc/olsrd/olsrd.conf
   ```

3. Prüfe, ob WireGuard läuft (OLSR benötigt das WireGuard-Interface):
   ```bash
   ip addr show wg0
   ```

4. Prüfe OLSR-Nachbarn:
   ```bash
   /usr/local/bin/neigh.sh
   ```

5. Erhöhe den Debug-Level in `/etc/olsrd/olsrd.conf`:
   ```
   DebugLevel 2
   ```

### fastd-Verbindungen funktionieren nicht

1. Prüfe, ob der Service läuft:
   ```bash
   systemctl status fastd@vpn
   ```

2. Prüfe die fastd-Konfiguration:
   ```bash
   cat /etc/fastd/vpn/fastd.conf
   ```

3. Prüfe die Logs:
   ```bash
   journalctl -u fastd@vpn -f
   ```

### Öffentliche Schlüssel abrufen

Nach dem Ausführen des Playbooks findest du den öffentlichen WireGuard-Schlüssel in:
```bash
cat /etc/wireguard/wg0_public.key
```

Dieser Schlüssel muss in der `wireguard_peer_public_key`-Variable der anderen Server verwendet werden.

## Referenzen

- [WireGuard Dokumentation](https://www.wireguard.com/)
- [OLSR Projekt](https://github.com/OLSR/olsrd)
- [Weimarer Freifunk VPN-Config Repository](https://github.com/weimarnetz/vpnconfig)
