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

### Phase 2 (aktuell implementiert)

- WireGuard-Installation und -Konfiguration für normale VPN-Server
- OLSR-Kompilierung aus dem Quellcode und Konfiguration
- Routing zwischen VPN-Servern über WireGuard-Interface
- iptables-Regeln für NAT-Masquerading (10.63.0.0/16, 10.64.0.0/16 -> !10.0.0.0/8)
- Rate-Limiting für UDP Port 698
- fastd für Router-Verbindungen mit automatischer Secret-Generierung
- Webserver (Nginx) mit statischem JSON-Endpoint (`/freifunk/vpn/`)

### Zukünftige Phasen

- Zentrale Server-Logik für WireGuard Policy Routing

## Voraussetzungen

- Ansible installiert (Version 2.9 oder höher)
- Zugriff auf die VPN-Server (SSH)
- sudo/root-Rechte auf den Servern
- Internet-Verbindung für Downloads (OLSR-Quellcode, Packages)

## Verwendung

### Inventory-Datei erstellen

Erstellen Sie eine Inventory-Datei (`inventory.ini` oder `inventory.yml`):

**Wichtige Ansible-Variablen:**
- `ansible_host`: IP-Adresse oder Hostname (nur wenn Inventory-Name != Verbindungsadresse)
- `ansible_port`: SSH-Port (Standard: 22, nur wenn abweichend)
- `ansible_user`: SSH-User (Standard: aktueller User, nur wenn abweichend)

```ini
[vpn_servers]
vpn-server1.example.com ansible_host=192.168.1.10
vpn-server2.example.com ansible_host=192.168.1.11
```

Oder in YAML-Format:

```yaml
vpn_servers:
  hosts:
    vpn-server1.example.com:
      ansible_host: 192.168.1.10
      wireguard_address: "10.0.1.1/24"
      wireguard_peers:
        - public_key: "PEER1_PUBLIC_KEY"
          allowed_ips: ["10.0.1.2/32", "10.0.2.0/24"]
          endpoint: "vpn-server2.example.com:51820"
    vpn-server2.example.com:
      ansible_host: 192.168.1.11
      wireguard_address: "10.0.1.2/24"
      wireguard_peers:
        - public_key: "PEER2_PUBLIC_KEY"
          allowed_ips: ["10.0.1.1/32", "10.0.1.0/24"]
          endpoint: "vpn-server1.example.com:51820"
```

### Variablen definieren

Sie können Variablen entweder in der Inventory-Datei, in einer separaten `group_vars/vpn_servers.yml` oder direkt im Playbook definieren:

```yaml
# inventory.yml oder group_vars/vpn_servers.yml
vpn_servers:
  hosts:
    vpn-server2:
      vpn_server_number: 2
      # WireGuard-IP wird automatisch: 10.63.1.5/30
      # WireGuard-Port wird automatisch: 51192
      # OLSR MainIP wird automatisch: 10.63.1.5
    vpn-server3:
      vpn_server_number: 3
      # WireGuard-IP wird automatisch: 10.63.1.9/30
      # WireGuard-Port wird automatisch: 51193
      # OLSR MainIP wird automatisch: 10.63.1.9

# Optional: Weitere OLSR-Einstellungen
olsr_hna4_networks:
  - "10.61.0.0 255.255.0.0"
```

### Playbook ausführen

```bash
ansible-playbook -i inventory.ini vpn-server-playbook.yml
```

### Outputs

Nach dem erfolgreichen Ausführen des Playbooks werden wichtige Informationen ausgegeben, ähnlich wie Terraform Outputs:

**WireGuard öffentlicher Schlüssel:**
Der öffentliche WireGuard-Schlüssel wird am Ende des Playbook-Laufs für jeden Server ausgegeben. Dieser Schlüssel muss an die Admins des zentralen Servers weitergegeben werden, damit der neue VPN-Server als Peer konfiguriert werden kann.

Beispiel-Output:
```
==========================================
WireGuard Setup abgeschlossen
==========================================
Server: vpn-server3.example.com (vpn_server_number: 3)

ÖFFENTLICHER SCHLÜSSEL (für zentralen Server):
abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ123456==

WireGuard Konfiguration:
  Interface: wg0
  Adresse: 10.63.1.9/30
  Port: 51193
  Endpoint: 77.87.48.19:51193

Bitte übermitteln Sie den öffentlichen Schlüssel an die
Admins des zentralen Servers, damit dieser konfiguriert werden kann.
==========================================
```

**fastd Secret:**
Das fastd Secret wird automatisch generiert (falls nicht vorhanden) und am Ende des Playbook-Laufs ausgegeben. Dieses Secret wird für die Router-Konfiguration benötigt.

**Hinweis:** Die Outputs werden immer angezeigt (auch bei wiederholten Playbook-Läufen ohne Änderungen), sodass die Schlüssel jederzeit verfügbar sind.

## Variablen

### WireGuard-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `vpn_server_number` | Server-Nummer (2-9) - **erforderlich** | - |
| `wireguard_interface` | Name des WireGuard-Interfaces | `wg0` |
| `wireguard_base_port` | Basis-Port (Server-Nummer wird hinzugefügt) | `51190` |
| `wireguard_ip_base` | Basis-IP für /30-Netze | `10.63.1` |
| `wireguard_peer_public_key` | **Wird beim Playbook-Lauf abgefragt** | - |
| `wireguard_peer_endpoint_host` | **Wird beim Playbook-Lauf abgefragt** | - |
| `wireguard_peer_allowed_ips` | AllowedIPs für Peer (optional) | Auto-generiert |
| `wireguard_persistent_keepalive` | Keep-Alive Intervall | `20` |

#### Automatische Berechnungen

Basierend auf `vpn_server_number` werden automatisch berechnet:

- **IP-Adresse**: Server 2 → `10.63.1.5/30`, Server 3 → `10.63.1.9/30`, etc.
  - Formel: `10.63.1.(server_num*4+1)/30`
- **Port**: Server 2 → Port endet auf 2 (z.B. `51192`), Server 3 → Port endet auf 3, etc.
  - Formel: `wireguard_base_port + server_number`
- **AllowedIPs** (Standard): Server-IP + `10.63.0.0/16` + `10.64.0.0/16`

#### Beispiel Inventory

```yaml
vpn_servers:
  hosts:
    vpn-server2:
      vpn_server_number: 2
      # IP wird automatisch: 10.63.1.5/30
      # Port wird automatisch: 51192
    vpn-server3:
      vpn_server_number: 3
      # IP wird automatisch: 10.63.1.9/30
      # Port wird automatisch: 51193
```

### Sensible Variablen (Verschlüsselt mit Ansible Vault)

**Wichtig:** Die folgenden Variablen **MÜSSEN** gesetzt werden (entweder im Inventory oder per Command-Line mit `-e`):
- `wireguard_peer_public_key`: Öffentlicher Schlüssel des WireGuard-Peers
- `wireguard_peer_endpoint_host`: Public IP-Adresse oder Hostname des Peers (Standard: `77.87.48.19`)
- `fastd_secret`: fastd Secret (privater Schlüssel)

**Möglichkeiten zum Setzen der Variablen:**

**Option 1: Im Inventory (unverschlüsselt - nicht empfohlen für sensible Daten)**

**INI-Format:**
```ini
[vpn_servers]
vpn-server3 vpn_server_number=3 wireguard_peer_public_key="PEER_PUBLIC_KEY_HIER" fastd_secret="fastd_secret_hier"

# Oder gemeinsam für alle Server:
[vpn_servers:vars]
wireguard_peer_public_key="PEER_PUBLIC_KEY_HIER"
fastd_secret="fastd_secret_hier"
```

**YAML-Format:**
```yaml
# inventory.yml
vpn_servers:
  hosts:
    vpn-server3:
      vpn_server_number: 3
      wireguard_peer_public_key: "PEER_PUBLIC_KEY_HIER"
      fastd_secret: "fastd_secret_hier"
```

**Option 2: Mit Ansible Vault verschlüsselt (empfohlen)**

1. **Einzelne Strings verschlüsseln:**
```bash
ansible-vault encrypt_string 'PEER_PUBLIC_KEY_HIER' --name 'wireguard_peer_public_key'
ansible-vault encrypt_string 'fastd_secret_hier' --name 'fastd_secret'
```

2. **In `group_vars/vpn_servers/vault.yml` speichern (empfohlen):**
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

3. **Direkt im YAML-Inventory (möglich, aber `group_vars` ist besser):**
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

**Hinweis:** Für INI-Inventory-Dateien ist dies nicht praktikabel, da mehrzeilige verschlüsselte Strings nicht gut unterstützt werden. **Empfehlung:** Verwenden Sie `group_vars/vpn_servers/vault.yml` auch bei INI-Inventory, da YAML-Dateien Vault-Syntax besser unterstützen.

**Option 3: Per Command-Line (nicht empfohlen für Produktion):**
```bash
# Unverschlüsselt
ansible-playbook -i inventory.yml vpn-server-playbook.yml \
  -e "wireguard_peer_public_key=PEER_KEY" \
  -e "fastd_secret=SECRET"

# Mit Vault (wenn andere Variablen verschlüsselt sind)
ansible-playbook -i inventory.yml vpn-server-playbook.yml \
  -e "wireguard_peer_public_key=PEER_KEY" \
  -e "fastd_secret=SECRET" \
  --ask-vault-pass
```

**Playbook mit Vault ausführen (wenn Variablen verschlüsselt sind):**
```bash
ansible-playbook -i inventory.yml vpn-server-playbook.yml --ask-vault-pass
```

**Hinweis:** Wenn die Variablen nicht gesetzt sind, schlägt das Playbook mit einer aussagekräftigen Fehlermeldung fehl.

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
| `fastd_gateway_ip_base` | Basis-IP für Gateway (wird als `{{ fastd_gateway_ip_base }}.{{ vpn_server_number }}` berechnet) | `10.63.0` |

**Automatische Berechnungen:**
- **Gateway-IP**: `10.63.0.<vpn_server_number>` (z.B. Server 2 → `10.63.0.2`)
- **Secret**: Wird automatisch mit `fastd -g` generiert und in `/etc/fastd/fastd_mesh_secret.key` gespeichert

### OLSR-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `olsr_branch` | Git-Branch des OLSR-Quellcodes (master/main) | `master` |
| `olsr_main_ip` | Main IP-Adresse des Servers (z.B. "10.63.1.9") | - |
| `olsr_wireguard_interface` | WireGuard-Interface für OLSR | `wg0` |
| `olsr_fastd_interface` | fastd-Interface für OLSR (optional) | `fastd_mesh` |
| `olsr_hna4_networks` | Liste von HNA4-Netzwerken (z.B. `["10.61.0.0 255.255.0.0"]`) | `[]` |
| `olsr_plugins` | Plugin-Konfigurationen (txtinfo, filtergw, jsoninfo, nameservice, arprefresh) | siehe defaults |

### VPN-Status-Variablen

| Variable | Beschreibung | Standard |
|----------|-------------|----------|
| `vpn_status_gateway_ip_base` | Basis-IP für Gateway (wird als `{{ vpn_status_gateway_ip_base }}.{{ vpn_server_number }}` berechnet) | `10.63.0` |
| `vpn_status_country` | Länder-Code für JSON | `DE` |
| `vpn_status_maxmtu` | MTU für JSON | `1280` |
| `vpn_status_port` | Port für JSON | `5001` |
| `vpn_status_clients` | Client-Anzahl für JSON | `0` |

**Automatische Berechnungen:**
- **Gateway-IP**: `10.63.0.<vpn_server_number>` (z.B. Server 2 → `10.63.0.2`)

## Rollen

### WireGuard-Rolle (`roles/wireguard/`)

- Installiert WireGuard und WireGuard-Tools
- Generiert private/public Key-Paare (falls nicht vorhanden)
- Erstellt WireGuard-Konfigurationsdatei (`/etc/wireguard/wg0.conf`)
- Aktiviert IP-Forwarding
- Startet und aktiviert den WireGuard-Service

**Generierte Dateien:**
- `/etc/wireguard/wg0_private.key` - Privater Schlüssel (nicht weitergeben!)
- `/etc/wireguard/wg0_public.key` - Öffentlicher Schlüssel (für Peer-Konfiguration)
- `/etc/wireguard/wg0.conf` - WireGuard-Konfiguration

**Wichtig:** Der öffentliche Schlüssel muss mit anderen Servern geteilt werden, um die Peer-Konfiguration zu vervollständigen.

### iptables-Rolle (`roles/iptables/`)

- Installiert `iptables-persistent` (Debian) oder `netfilter-persistent` (RedHat)
- Erstellt persistente iptables-Regeln in `/etc/iptables/rules.v4`
- NAT-Masquerading für `10.63.0.0/16` und `10.64.0.0/16` -> `!10.0.0.0/8`
- Rate-Limiting für UDP Port 698 (200/sec, burst 200), dann DROP
- Aktiviert `netfilter-persistent` Service für automatisches Laden beim Boot

**Generierte Dateien:**
- `/etc/iptables/rules.v4` - iptables-Regeln

### fastd-Rolle (`roles/fastd/`)

- Installiert fastd
- Generiert Secret automatisch mit `fastd --generate-key` (falls nicht vorhanden)
- Erstellt fastd-Konfigurationsdatei (`/etc/fastd/vpn/fastd.conf`)
- Gateway-IP wird automatisch als `10.63.0.<vpn_server_number>` berechnet
- Erstellt `/etc/fastd/vpn/peers/` Verzeichnis für Peer-Konfigurationen
- Erstellt und aktiviert Systemd-Service (`fastd@fastd_mesh.service`)
- Gibt fastd Secret im Playbook-Output aus

**Generierte Dateien:**
- `/etc/fastd/vpn/secret.key` - fastd Secret (nicht weitergeben!)
- `/etc/fastd/vpn/fastd.conf` - fastd-Konfiguration
- `/etc/fastd/vpn/peers/` - Verzeichnis für Peer-Konfigurationen
- `/etc/systemd/system/fastd@fastd_mesh.service` - Systemd-Service-Datei

**Wichtig:** Das fastd Secret wird im Playbook-Output ausgegeben und kann für Router-Konfigurationen verwendet werden.

### OLSR-Rolle (`roles/olsr/`)

- Installiert Build-Dependencies (gcc, make, libnl-dev, etc.)
- Lädt OLSR-Quellcode von GitHub herunter (vom master/main Branch)
- Ermittelt Git-Hash des heruntergeladenen Quellcodes
- Prüft aktuelle OLSR-Version auf dem Server (falls installiert)
- Kompiliert OLSR nur wenn nötig: `make && make libs`
- Installiert OLSR nur wenn Version sich unterscheidet: `make install && make install_libs`
- Erstellt OLSR-Konfigurationsdatei (`/etc/olsrd/olsrd.conf`) mit allen Plugins
- Erstellt und aktiviert Systemd-Service

**Versionsprüfung:** Die Rolle vergleicht den Git-Hash des heruntergeladenen Quellcodes mit der installierten Version. Nur wenn sie sich unterscheiden oder OLSR nicht installiert ist, wird neu kompiliert und installiert.

**Generierte Dateien:**
- `/etc/olsrd/olsrd.conf` - OLSR-Konfiguration
- `/etc/systemd/system/olsrd.service` - Systemd-Service-Datei

### VPN-Status-Rolle (`roles/vpn-status/`)

- Installiert Nginx
- Erstellt Verzeichnis `/var/www/html/freifunk/vpn/`
- Erstellt statische JSON-Datei (`/var/www/html/freifunk/vpn/index.json`)
- Gateway-IP wird automatisch als `10.63.0.<vpn_server_number>` berechnet
- Konfiguriert Nginx Location-Block für `/freifunk/vpn/`
- Aktiviert und startet Nginx-Service

**Generierte Dateien:**
- `/var/www/html/freifunk/vpn/index.json` - Statische JSON-Datei mit Server-Status

## Troubleshooting

### WireGuard-Verbindung funktioniert nicht

1. Prüfen Sie, ob der Service läuft:
   ```bash
   systemctl status wg-quick@wg0
   ```

2. Prüfen Sie die WireGuard-Konfiguration:
   ```bash
   wg show
   ```

3. Prüfen Sie die Logs:
   ```bash
   journalctl -u wg-quick@wg0 -f
   ```

4. Stellen Sie sicher, dass die öffentlichen Schlüssel auf beiden Seiten korrekt sind

### OLSR findet keine Peers

1. Prüfen Sie, ob der Service läuft:
   ```bash
   systemctl status olsrd
   ```

2. Prüfen Sie die OLSR-Konfiguration:
   ```bash
   cat /etc/olsrd/olsrd.conf
   ```

3. Prüfen Sie, ob WireGuard läuft (OLSR benötigt das WireGuard-Interface):
   ```bash
   ip addr show {{ olsr_wireguard_interface | default('wg0') }}
   ```

4. Erhöhen Sie den Debug-Level in `/etc/olsrd/olsrd.conf`:
   ```
   DebugLevel 2
   ```

### Öffentliche Schlüssel für Peer-Konfiguration abrufen

Nach dem ersten Ausführen des Playbooks finden Sie den öffentlichen Schlüssel in:
```bash
cat /etc/wireguard/wg0_public.key
```

Dieser Schlüssel muss in der `wireguard_peers`-Konfiguration der anderen Server verwendet werden.

## Entwicklung

### Rollen-Struktur

```
roles/
  ├── wireguard/
  │   ├── tasks/main.yml        # Haupt-Tasks
  │   ├── handlers/main.yml     # Service-Handler
  │   └── templates/
  │       └── wg0.conf.j2       # WireGuard-Config Template
  ├── iptables/
  │   ├── tasks/main.yml        # Haupt-Tasks
  │   ├── handlers/main.yml     # Handler für Regel-Anwendung
  │   └── templates/
  │       └── iptables-rules.v4.j2  # iptables-Regeln Template
  ├── olsr/
  │   ├── tasks/main.yml        # Haupt-Tasks
  │   ├── handlers/main.yml     # Service-Handler
  │   └── templates/
  │       ├── olsrd.conf.j2     # OLSR-Config Template
  │       └── olsrd.service.j2  # Systemd-Service Template
  ├── fastd/
  │   ├── tasks/main.yml        # Haupt-Tasks
  │   ├── handlers/main.yml     # Service-Handler
  │   └── templates/
  │       ├── fastd_mesh.conf.j2    # fastd-Config Template
  │       └── fastd.service.j2      # Systemd-Service Template
  └── vpn-status/
      ├── tasks/main.yml        # Haupt-Tasks
      └── templates/
          └── status.json.j2    # JSON-Status Template
```

### Erweiterungen

Um neue Funktionen hinzuzufügen:

1. Neue Rollen in `roles/` erstellen
2. Rollen im Hauptplaybook (`vpn-server-playbook.yml`) hinzufügen
3. Dokumentation in diesem README aktualisieren

## Referenzen

- [WireGuard Dokumentation](https://www.wireguard.com/)
- [OLSR Projekt](https://github.com/OLSR/olsrd)
- [Weimarer Freifunk VPN-Config Repository](https://github.com/weimarnetz/vpnconfig)

