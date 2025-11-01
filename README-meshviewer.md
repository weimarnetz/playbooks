# Meshviewer Ansible Playbook

Dieses Ansible Playbook installiert und konfiguriert automatisch Meshviewer auf einem Server.

## Voraussetzungen

- Ansible installiert
- Zielserver mit Benutzer `hopglass` (bereits vorhanden)
- Webserver bereits konfiguriert
- Python3 und pip verfügbar
- Git installiert

## Verwendung

1. **Inventory-Datei erstellen:**
   ```bash
   echo "your-server-ip ansible_user=your-username" > inventory.ini
   ```

2. **Playbook ausführen:**
   ```bash
   ansible-playbook -i inventory.ini meshviewer-playbook.yml
   ```

## Was wird installiert

- **Meshviewer**: Neueste Version von GitHub heruntergeladen nach `/home/hopglass/meshviewer`
- **Device-Pictures**: Repository geklont nach `/home/hopglass/meshviewer/device-pictures`
- **owm2meshviewer**: Python-Skript mit virtualenv in `/home/hopglass/owm2meshviewer`
- **Cron-Job**: Ausführung alle 10 Minuten
- **Konfiguration**: `config.json` nach `/home/hopglass/meshviewer/`

## Verzeichnisstruktur nach Installation

```
/home/hopglass/
├── meshviewer/
│   ├── (Meshviewer-Dateien)
│   ├── device-pictures/
│   └── config.json
└── owm2meshviewer/
    ├── owm2meshviewer.py
    └── venv/
```

## Cron-Job

Der Cron-Job wird automatisch für den Benutzer `hopglass` eingerichtet und führt das owm2meshviewer-Skript alle 10 Minuten aus.

## Anpassungen

Falls Anpassungen an der Konfiguration nötig sind, können die Dateien in `roles/meshviewer/files/` bearbeitet werden:
- `meshviewer_config.json` - Meshviewer-Konfiguration
- `owm2meshviewer.py` - Python-Skript für Datenverarbeitung
