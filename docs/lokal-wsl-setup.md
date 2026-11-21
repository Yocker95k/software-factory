# Software Factory lokal unter Windows einrichten (WSL2)

Schritt für Schritt vom Windows-Rechner bis zur laufenden Factory, ohne eigenen Server.

> 🚧 **Experimentell.** Die Factory ist für einen Linux-Server gebaut und dort getestet ([Anleitung für den VPS](hostinger-vps-setup.md)). Die lokale Variante eignet sich zum Ausprobieren. Die Factory arbeitet nur, solange dein Rechner läuft und nicht im Ruhezustand ist.

**So funktioniert es:** Mit WSL2 läuft ein echtes Ubuntu in Windows. Darin installierst du Docker und die Factory genauso wie auf einem Server. Jeder Agent-Lauf bekommt weiterhin seinen eigenen Container.

**Dauer:** etwa 30–45 Minuten, davon viel Warten auf Installationen.

**Du brauchst:**

- Windows 10 (ab 22H2) oder Windows 11 mit aktivierter Virtualisierung (bei aktuellen Rechnern Standard)
- mindestens 16 GB RAM im Rechner, davon 8 GB für die Factory
- ein Claude-Abo (Pro, Max, Team oder Enterprise) **oder** einen Anthropic-API-Key
- einen GitHub-Account
- unter Windows: [Claude Code](https://code.claude.com), `git` und die [GitHub CLI](https://cli.github.com) (`gh`)

> Docker Desktop brauchst du nicht. Docker läuft direkt in Ubuntu. Hast du Docker Desktop schon installiert, schalte dort die WSL-Integration für Ubuntu aus, damit sich die beiden nicht in die Quere kommen.

---

## 1. Ubuntu installieren

Öffne PowerShell **als Administrator**:

```powershell
wsl --install -d Ubuntu-24.04
```

Starte den Rechner neu. Öffne dann **Ubuntu 24.04** aus dem Startmenü und leg einen Linux-Benutzernamen und ein Passwort an.

## 2. Speicher und Dauerbetrieb einstellen

Leg unter Windows die Datei `.wslconfig` in deinem Benutzerordner an (`C:\Users\DEIN-NAME\.wslconfig`):

```ini
[wsl2]
memory=8GB
processors=4

[general]
# WSL nicht beenden, wenn kein Ubuntu-Fenster offen ist
vmIdleTimeout=-1
```

Stell außerdem in den Windows-Energieeinstellungen ein, dass der Rechner im Netzbetrieb nicht in den Ruhezustand geht.

## 3. systemd einschalten

Die Factory läuft als Dienst und braucht dafür systemd. Gib in Ubuntu ein:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Führ danach in PowerShell `wsl --shutdown` aus und öffne Ubuntu wieder. Zum Prüfen: `systemctl is-system-running` meldet `running` oder `degraded`.

## 4. Factory installieren

Alle weiteren Befehle in Ubuntu laufen als root, denn der Dienst läuft als root und nutzt dessen GitHub-Login:

```bash
sudo -i
git clone https://github.com/AlexPEClub/software-factory.git /opt/software-factory
cd /opt/software-factory
./setup.sh
```

> **Wichtig:** Die Factory muss im Linux-Dateisystem liegen (`/opt/...`), **nicht** unter `/mnt/c/` oder `/mnt/d/`. Auf den Windows-Laufwerken ist der Dateizugriff sehr langsam, und die Factory kann dort keine Dateirechte setzen.

`setup.sh` erkennt WSL und stellt die Vorschau auf `localhost` ein.

## 5. Anmelden und starten

Weiter als root in Ubuntu:

1. **Claude:** Führ unter Windows `claude setup-token` aus. Trag den Token als `CLAUDE_CODE_OAUTH_TOKEN=...` in `/etc/software-factory/secrets.env` ein (`nano /etc/software-factory/secrets.env`).
2. **GitHub:** `gh auth login -s project`
3. **Lauf-Anzahl:** In `/etc/software-factory/config.json` den Wert `"maxParallel": 1` setzen. Jeder Lauf darf bis zu 3 GB Speicher belegen.
4. **Starten:** `systemctl restart software-factory`

## 6. Projekt anbinden

`scripts/new-project.sh` verbindet sich per SSH mit einem Server. Lokal brauchst du das nicht. Leg das Repo stattdessen direkt in Ubuntu an:

```bash
# in Ubuntu als root
gh repo create DEIN-GITHUB-NAME/mein-projekt --private --template AlexPEClub/factory-template
cd /opt/software-factory && ./scripts/add-repo.sh DEIN-GITHUB-NAME/mein-projekt
```

Ein bestehendes Repo bindest du nur mit `add-repo.sh` an.

Unter Windows arbeitest du dann wie gewohnt weiter:

```powershell
gh repo clone DEIN-GITHUB-NAME/mein-projekt
```

Installier einmalig die lokalen Skills (`./scripts/install-local-skills.sh` aus einem Klon dieses Repos, z. B. in Git Bash). Danach geht es im Projekt weiter mit dem Briefing in `docs/briefing.md`, dann `/init`, dann pro Feature `/write-spec` und `/to-factory`.

## 7. Prüfen, ob alles läuft

```bash
systemctl status software-factory      # Dienst aktiv?
journalctl -u software-factory -f      # Log, sollte "runner:started" zeigen
docker ps                              # laufende Agent-Container
```

Vorschauen öffnest du im Windows-Browser unter `http://localhost:4001`. Der Login ist `factory`, das Passwort steht in `/etc/software-factory/config.json`. WSL leitet die Ports automatisch an Windows weiter.

## Häufige Probleme

- **`setup.sh` meldet „systemd läuft nicht“:** Schritt 3 wiederholen. Danach unbedingt `wsl --shutdown` ausführen.
- **Die Factory steht nach einer Weile still:** Der Rechner war im Ruhezustand oder WSL wurde beendet. Prüf `vmIdleTimeout` und die Energieeinstellungen aus Schritt 2.
- **Die Vorschau ist unter Windows nicht erreichbar:** Führ `wsl --shutdown` aus und öffne Ubuntu neu. Prüf, dass in `.wslconfig` nicht `localhostForwarding=false` steht.
- **Container brechen mit Speicherfehlern ab:** Erhöh in `.wslconfig` den Wert `memory` oder lass `maxParallel` auf `1`.
