# 05 – Monitoring & Compliance-Baseline

**Version:** 1.0  
**Phase:** 2 (Monitoring-Basics), 3 (Advanced Observability)  
**Säule:** B (Maschinenraum), aber A-sichtbar  
**Datum:** 2025-11-19  

---

## 🧭 Worum es bei Monitoring & Compliance in Ground-Zero geht

Monitoring & Compliance sind bei dir keine „Add-Ons", sondern ein eigener Kernstrang der Infrastruktur, der in den Infra-Specs und Phase-Docs explizit als Pflicht definiert ist.

Die Dateien beschreiben ein System, das schon in Phase 2 eine **Minimal-Überwachung und Compliance-Checks** via Logs, Cron-Healthchecks und einem System-Audit-Script hat und in Phase 3 zu einer **vollwertigen Observability-Plattform** mit Prometheus, Grafana, Sentry und Alerting ausgebaut wird.

Im gleichen Atemzug verankerst du Compliance-Aspekte – Konfigurations-Checklisten, Diagnostics-Baselines und ein wöchentliches Validation-Script mit einem **Pass-Threshold von 90 %** – als festen Teil der Infrastruktur, nicht als „man schaut halt gelegentlich in Logfiles".

Damit ist Monitoring/Compliance eine **eigene Achse neben MCP, Maschinenraum und DR**: sie sagt dir, ob der Zustand des Systems überhaupt noch dem entspricht, was in Projektdaten/Architektur versprochen wurde.

### 🔁 Veranschaulichung – Monitoring & Compliance im Gesamtbild

```text
        +-------------------------------+
        |  Säule A: Desktop + MCP       |
        |  - Startet Audits/Checks      |
        +---------------+---------------+
                        |
                        v
        +-------------------------------+
        |  Säule B: Maschinenraum       |
        |  - Logs, Healthchecks, Cron   |
        |  - Backup & DR-Skripte        |
        +---------------+---------------+
                        |
                        v
        +-------------------------------+
        |  Monitoring & Compliance      |
        |  - Specs: n8n-postgres-stack  |
        |  - Specs: monitoring-basics   |
        |  - system-audit.sh, pg-backup |
        +-------------------------------+
```

---

## 🧩 Was in deinen Monitoring-/Compliance-Specs tatsächlich drinsteckt

In den langen YAML-/Spec-Blöcken, die du generiert hast, steckt deutlich mehr als „wir loggen ein bisschen"; hier ein präziser Auszug, damit klar ist, was **schon beschrieben** ist.

- **Logs**: System-Logs (`/var/log/syslog`), Auth-Logs (`/var/log/auth.log`), UFW-Firewall-Logs (`/var/log/ufw.log`), Docker-Daemon-Logs, Postgres-Container-Logs, n8n-Logs, Caddy-Logs – jeweils mit Retention-Zeiten und Threshold-Checks.
- **Healthchecks**: Checks für `docker is-active`, Diskusage (z.B. `df` mit 85 %-Schwellwert), RAM-Usage, CPU-Last, Container-Health (Postgres/N8N/Caddy), Dienststatus (UFW, Fail2ban).
- **Backups**: Cron-Job für `pg-backup.sh` um 04:00, Wochen-Backups für Volumes, RTO/RPO-Ziele dokumentiert, zugehöriges `pg-restore.sh`.
- **Compliance-Checks**: `system-audit.sh` als wöchentliches Validation-Script mit definiertem Pass-Threshold (90 % der Checks müssen „PASS" liefern).
- **Alerts & Channels**: Email und optional Webhook-Alert-Kanäle, mit Regeln für `servicedown`, `diskcritical`, `backupfailed`, `highload`, inkl. Throttling.
- **Lifecycle**: Angaben zur erwarteten Server-Lebenszeit, Upgrade-Zyklen, Rebuild-Triggern (OpenBao-Migration, Major-Incident, Performance-Degradation).

### 📊 Veranschaulichung – Inhaltsmatrix deiner Specs

| Kategorie | Beispiele aus deinen Specs | Bedeutung |
|----------|----------------------------|-----------|
| Logs | `syslog`, `auth.log`, `ufw.log`, Docker-Logs, n8n/Caddy/Postgres-Logs | Basis für Fehleranalyse & Security-Audits |
| Healthchecks | `dockerrunning`, `diskusage`, `ramusage`, Container-Health | Frühwarnsystem für Ressourcen/Services |
| Backups | `pg-backup.sh` (daily), Volume-Backups (weekly) | DR-Fundament, gekoppelt mit RTO/RPO |
| Compliance | Validation-Script, Pass-Threshold 90 %, F1-/D1-Baseline | Sicherstellung von Konfig-/Diagnose-Konformität |
| Alerts | Email/Webhook-Channels, Rules für Ausfälle/Last/Backups | Automatisierte Kommunikation bei Problemen |
| Lifecycle | Lifetime 12 Monate, Upgrades alle 3 Monate | Planbarer Server-Lebenszyklus |

---

## 🔭 Monitoring-Phasen: Minimal vs. Advanced

In `monitoring-basics` und dem Infra-Spec-Text definierst du zwei klar getrennte Monitoring-Stufen:

1. **Phase 2 – Minimal Monitoring**: Fokus auf Logs, Cron-Healthchecks, einfache Shell-Metriken (CPU/RAM/Disk, Container-Count, Postgres-Connections).
2. **Phase 3 – Advanced Monitoring**: Erweiterung um Prometheus, Grafana, Sentry, Loki, Exporter (node_exporter, postgres_exporter, cadvisor), Alertmanager/PagerDuty.

Der Clou: Schon in Phase 2 sind die Checks so formuliert, dass sie später relativ leicht in Prometheus-Regeln/Alerts überführt werden können; die Shell-Kommandos sind im Prinzip nur die „manuelle" Version dessen, was später Metrics/Exporter tun.

### 📊 Veranschaulichung – Vergleich Phase 2 vs. Phase 3

| Aspekt | Phase 2 (Minimal) | Phase 3 (Advanced) |
|--------|-------------------|--------------------|
| Logs | journald + Files (`syslog`, `auth.log`, Docker) | plus zentrale Log-Aggregation (Loki/ELK) |
| Metriken | Shell-Kommandos (CPU, RAM, Disk via `top/free/df`) | Prometheus-Exporter, Dashboards in Grafana |
| Alerts | Email + optional Webhook, Cron-basiert | Alertmanager, ggf. PagerDuty, strukturierte Alert-Rules |
| Checks | Cron-Skripte (`system-audit.sh`, Disk-Checks) | Prometheus-Rules, SLO-basierte Alerts |

---

## 📜 Konkrete Log- und Healthcheck-Konfiguration

Die Specs gehen so weit, dass sie Pfade, Retention-Dauern, Thresholds und Kommandos definieren – das ist viel „dicker", als es auf den ersten Blick wirkt.

### System-Logs

- `syslog`: 30 Tage Retention, rotierend, max 100 MB, max 5 Files.
- `auth.log`: 90 Tage Retention, basis für SSH-/Auth-Analysen.
- `ufw.log`: 7 Tage Retention, Monitoring auf geblockte Verbindungen.

Dazu Checks wie: Anzahl Errors im Syslog in der letzten Stunde, Anzahl „Failed password" in `auth.log`, Anzahl „BLOCK"-Einträge im UFW-Log.

### Docker-Logs

- Docker-Daemon-Logs via journald, 7 Tage Retention.
- Postgres-Logs: FATAL/ERROR-Count in der letzten Stunde mit Schwellwert.
- n8n-Logs: Error-Count in den letzten 5/60 Minuten.
- Caddy-Logs: TLS/Certificate-Fehler in einem gewissen Zeitraum.

### 📋 Veranschaulichung – Ausschnitt (vereinfacht)

```yaml
system:
  - name: syslog
    path: /var/log/syslog
    retentiondays: 30
  - name: authlog
    path: /var/log/auth.log
    retentiondays: 90
docker:
  - name: postgrescontainer
    path: docker logs postgres
    retentiondays: 7
  - name: n8ncontainer
    path: docker logs n8n
    retentiondays: 7
```

ASCII-Bild:

```text
[System] -> syslog/auth.log/ufw.log
 [Docker] -> daemon + postgres/n8n/caddy logs
    |
    +--> Cron-Skripte / system-audit.sh -> Threshold-Checks
           |
           +--> Alerts (Email/Webhook)
```

---

## ⚙️ Healthcheck- und Metric-Details

`monitoring-basics` definiert eine ganze Batterie von Checks, die regelmäßig laufen sollen:

- Docker-Daemon aktiv? (`systemctl is-active docker`) – alle 60 Sekunden.
- UFW aktiv? (`ufw status | grep Status: active`) – alle 300 Sekunden.
- Fail2ban aktiv? – optional, aber vorgesehen.
- Container-Health: Postgres „healthy", n8n „running", Caddy „running".
- Ressourcen: CPU-Usage, RAM-Usage, Disk-Usage per Shell, CPU-Load, Docker-Container-Anzahl, Postgres-Connection-Count.

Zusätzlich definierst du Cron-Zeitpläne, z.B. Healthcheck alle 10 Minuten, Diskcheck stündlich, Backup-Job täglich 04:00, etc.

### 📊 Veranschaulichung – Beispiel Healthcheck-Matrix

| Check | Command | Threshold | Frequenz |
|-------|---------|-----------|----------|
| Docker läuft? | `systemctl is-active docker` | `active` | alle 60 s |
| Disk-Usage | `df` / `awk` | 85 % | alle 300 s |
| RAM-Usage | `free` / `awk` | 85 % | alle 300 s |
| Postgres-Connections | `psql`/`pg_stat_activity` | 95 % der Max-Conn | alle 15 min |
| n8n Errors | `docker logs n8n --since 5m` | ≥ 10 Fehler | alle 5 min |

---

## ✔️ Compliance-Baseline & Validation-Script

In deinem Spec ist ein **Compliance-Block** definiert, der Ground-Zero-Phase-2-Baseline, F1-Configuration-Checklist, D1-Diagnostics-Baseline und ein wöchentliches Validation-Script benennt.

Das Script `system-audit.sh` (Pfad `opt/claude-agents/scripts/analysis/system-audit.sh`) soll wöchentlich laufen und einen Pass-Threshold von **90 %** haben – ist weniger erfüllt, gilt der Server als „nicht compliant".

Das verschiebt Compliance von „weiches Bauchgefühl" zu einer objektiven, messbaren Checkliste, die automatisiert evaluiert werden kann.

### 📋 Veranschaulichung – Compliance-Snippet (aus Spec-Text)

```yaml
compliance:
  standards:
    - "Ground-Zero Phase 2 Baseline"
    - "F1 Configuration Checklist"
    - "D1 Diagnostics Baseline"
  validation:
    script: /opt/claude-agents/scripts/analysis/system-audit.sh
    frequency: weekly
    passthreshold: 90
```

ASCII-Flow:

```text
[system-audit.sh weekly]
        |
        v
  Checks (Config, Security, Monitoring)
        |
        +--> PASS >= 90% -> "OK"
        |
        +--> PASS < 90%  -> Alert + ToDo-Liste
```

---

## 📣 Alerts, Channels & Regeln

Deine Monitoring-Spec beschreibt zwei Alert-Kanäle: **Email** (root@localhost) und optional einen **Webhook**, der z.B. Slack/Discord oder ein anderes System bedienen kann.

Darauf aufbauend definierst du fertige Alert-Regeln mit Severity, Bedingungen und Throttling, z.B. `servicedown`, `diskcritical`, `backupfailed`, `highload`.

### 📋 Veranschaulichung – Alerts-Ausschnitt

```yaml
channels:
  - name: email
    type: email
    enabled: true
    config:
      recipient: root@localhost
      subjectprefix: "Ground-Zero Alert"
  - name: webhook
    type: webhook
    enabled: false
    config:
      url: ${MONITORING_WEBHOOK_URL}
      method: POST

rules:
  - name: servicedown
    severity: critical
    condition: containerstatus != "running"
    channels: [email, webhook]
    throttleminutes: 15
  - name: backupfailed
    severity: warning
    condition: backupexitcode != 0
    channels: [email]
    throttleminutes: 1440
```

Das heißt: Ein gescheitertes Backup führt nicht nur zu einem Log, sondern zu einem Alert mit klar definiertem Versandverhalten (maximal 1x am Tag, damit du nicht zugespammt wirst).

---

## 🔄 Server-Lifecycle & Compliance-Einbettung

Dein Infra-Spec enthält auch einen **Lifecycle-Block**, der sagt: erwartete Lebenszeit 12 Monate, OS-Upgrades alle 3 Monate, Rebuild-Trigger bei großen Technologie-Sprüngen (z.B. OpenBao-Migration, Kubernetes), Major-Incidents oder anhaltender Performance-Degradation trotz Optimierung.

Das ist wichtig, damit eine KI nicht davon ausgeht, dass ein einmal aufgesetzter Droplet „für immer" läuft, sondern versteht, dass Rebuilds Teil der Strategie sind – mit Monitoring & Compliance als Signale, wann es Zeit für einen Rebuild ist.

### 🔁 Veranschaulichung – Lifecycle-Logik

```text
Zeit / Ereignisse
   |
   |-- Regelmäßige Updates (alle 3 Monate)
   |-- Weekly system-audit.sh (Compliance >= 90%?)
   |-- Monitoring-Alerts (servicedown, diskcritical, backupfailed)
   |
   +--> Wenn:
         - Major Incident
         - Compliance dauerhaft < 90%
         - Performance schlecht trotz Optimierung
       => Rebuild/Upgrade des Droplets
```

---

## 🧠 Warum das jetzt weniger missverständlich ist

Viele dieser Dinge sind in deinen Specs **extrem kompakt** als YAML/Listen beschrieben: „autoupdates enabled true", „backups retentiondays 7", „compliance passthreshold 90", aber nicht, was das in der Praxis und im Zusammenspiel bedeutet.

Jetzt ist explizit nachgezogen: welche Logs, welche Checks, welche Skripte, welche Ziele, welche Alerts – und wie alles mit GDPR/SLA/DR-Zielen und Agent-/Skill-Ebenen verknüpft ist.

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige Monitoring-/Compliance-Beschreibung |

---

*Lies als nächstes `06-Skills-und-Agents-Orchestrierung.md` für Details zum Skill-Ökosystem.*
