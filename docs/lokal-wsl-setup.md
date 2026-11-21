# Factory lokal unter Windows betreiben (WSL2)

Statt auf einem VPS kannst du die Factory auf deinem Windows-Rechner laufen lassen. Dafür nutzt du **WSL2**, ein echtes Ubuntu in Windows. Darin laufen Docker und die Factory so wie auf einem Server.

> **Gut zu wissen:** Die Factory arbeitet nur, solange dein Rechner an ist und nicht schläft. Ein VPS läuft rund um die Uhr, der Rechner zu Hause nicht. Zum Ausprobieren reicht die lokale Variante.

## Warum WSL2 und nicht DevContainer, Sandbox oder Docker Desktop?

- **WSL2 + Docker in Ubuntu:** Die Factory ist für Ubuntu mit systemd gebaut. `setup.sh` läuft damit fast unverändert.
- **DevContainer:** Damit entwickelst du *in* einem Container, z. B. in VS Code. Das brauchst du nur, wenn du am Runner selbst programmierst, nicht für den Betrieb.
- **Sandbox (`/sandbox` in Claude Code):** Die schützt deine eigenen Claude-Sitzungen auf dem Rechner (`/write-spec`, `/to-factory`). Die Factory-Agents sind schon abgeschottet: Jeder Lauf hat einen eigenen Container mit Speicher- und Prozesslimit, die Vorschau läuft ohne Zugangsdaten.
- **Runner selbst in Docker:** Der Runner startet Container und gibt ihnen Ordner vom Host mit. In einem Container bräuchte er vollen Zugriff auf Docker. Das macht es nur komplizierter und nicht sicherer.
- **Docker Desktop:** Es geht auch, wenn du die WSL-Integration aktivierst. Docker direkt in Ubuntu ist aber näher am Server-Aufbau. Installiere nicht beides gleichzeitig.

## Voraussetzungen

- Windows 10/11 mit aktivierter Virtualisierung (Standard bei aktuellen Rechnern)
- Mindestens 16 GB RAM im Rechner, damit 8 GB für die Factory übrig bleiben
- Ein Claude-Abo oder ein Anthropic-API-Key, ein GitHub-Account

## 1. Ubuntu installieren

In PowerShell **als Administrator**:

```powershell
wsl --install -d Ubuntu-24.04
```

Neu starten, Ubuntu aus dem Startmenü öffnen und einen Linux-Benutzernamen und ein Passwort anlegen.

## 2. Speicher und Dauerbetrieb einstellen

Lege unter Windows die Datei `%UserProfile%\.wslconfig` an (z. B. `C:\Users\DEINNAME\.wslconfig`):

```ini
[wsl2]
memory=8GB
processors=4

[general]
# WSL nicht herunterfahren, wenn kein Fenster offen ist
vmIdleTimeout=-1
```

Stell außerdem in den Windows-Energieoptionen ein, dass der Rechner im Netzbetrieb nicht in den Ruhezustand geht.

## 3. systemd aktivieren

In Ubuntu:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Danach in PowerShell `wsl --shutdown` und Ubuntu wieder öffnen. Prüfen: `systemctl is-system-running` gibt `running` oder `degraded` aus.

## 4. Factory installieren

Alles Weitere als root in Ubuntu. Der Dienst läuft als root und nutzt dessen GitHub-Login.

```bash
sudo -i
git clone https://github.com/AlexPEClub/software-factory.git /opt/software-factory
cd /opt/software-factory
./setup.sh
```

> **Wichtig:** Die Factory gehört ins Linux-Dateisystem (`/opt/...`), **nicht** nach `/mnt/c/` oder `/mnt/d/`. Dort ist der Dateizugriff sehr langsam und Besitzrechte lassen sich nicht setzen.

`setup.sh` erkennt WSL und trägt als Vorschau-Adresse `localhost` ein.

## 5. Logins und erstes Repo

Weiter als root in Ubuntu:

1. **Claude:** unter Windows `claude setup-token` ausführen und den Token als `CLAUDE_CODE_OAUTH_TOKEN=...` in `/etc/software-factory/secrets.env` eintragen (`nano /etc/software-factory/secrets.env`)
2. **GitHub:** `gh auth login -s project`
3. **Starten:** `systemctl restart software-factory`

In `/etc/software-factory/config.json` empfehle ich lokal `"maxParallel": 1`. Jeder Lauf darf bis zu 3 GB belegen.

**Repo anbinden:** `new-project.sh` verbindet sich per SSH mit einem Server, das brauchst du lokal nicht. Stattdessen:

```bash
# in Ubuntu als root
gh repo create DEIN-NAME/mein-projekt --private --template AlexPEClub/factory-template
cd /opt/software-factory && ./scripts/add-repo.sh DEIN-NAME/mein-projekt
```

Das Projekt klonst du dann unter Windows ganz normal (`gh repo clone DEIN-NAME/mein-projekt`). Dort arbeitest du mit `/init`, `/write-spec` und `/to-factory` weiter (lokale Skills mit `./scripts/install-local-skills.sh` installieren).

## 6. Prüfen

```bash
systemctl status software-factory      # aktiv?
journalctl -u software-factory -f      # "runner:started"?
docker ps                              # laufende Jobs
```

Vorschauen öffnest du unter Windows im Browser unter `http://localhost:4001` (Login `factory`, Passwort in `/etc/software-factory/config.json`). WSL leitet die Ports automatisch an Windows weiter.

## Häufige Probleme

- **„systemd läuft nicht“:** Schritt 3 wiederholen und dabei `wsl --shutdown` nicht vergessen.
- **Factory steht nach einer Weile still:** Rechner schläft oder WSL wurde beendet. Siehe `vmIdleTimeout` und Energieoptionen in Schritt 2.
- **Vorschau unter Windows nicht erreichbar:** `wsl --shutdown`, Ubuntu neu öffnen. Prüfen, ob in `.wslconfig` `localhostForwarding` nicht auf `false` steht.
