# 04 – Backup & DR-Stack: Dein Lebensversicherung

**Version:** 1.0  
**Phase:** 2 (Server-Stack + DR-Basics)  
**Säule:** B (Maschinenraum), aber A-gesteuert  
**Datum:** 2025-11-19  

---

## 💾 Worum es bei Backup & DR in Ground-Zero geht

Ground-Zero hängt an einem einzigen, bewusst stark gemachten **PostgreSQL-16-Cluster**; alles, was n8n, Agent-Zero, E2B-Runs oder spätere Analytics betrifft, landet letztlich dort, also ist DR nicht „extra", sondern Überlebensgrundlage.

Deine Architektur-Docs machen klar: Ziel ist eine **3-2-1-Backup-Strategie** mit täglicher Sicherung, verschlüsselten Offsite-Backups und klaren RTO/RPO-Zielen, die realistisch testbar sind.

### 🔁 Veranschaulichung – Backup-Landschaft (ASCII)

```text
                 +---------------------+
                 |   Säule B: Droplet  |
                 |  postgres:16-alpine |
                 +----------+----------+
                            |
                 (pg_dump / pg_basebackup)
                            |
        +-------------------+-------------------+
        |                                       |
+---------------+                       +---------------+
| Local Backup  |                       | Offsite Store |
| /backups/...  |                       | (z.B. S3/DO)  |
+-------+-------+                       +-------+-------+
        |                                        |
        +--------[GPG / Encryption]-------------+
```

---

## 🎯 Ziele: RTO, RPO & 3-2-1-Regel

Deine Projekt-/Architektur-Docs formulieren kein vages „wir sollten Backups machen", sondern präzise Targets: **RTO < 4 h**, **RPO < 24 h**, abgesichert durch tägliche Dumps und regelmäßige Restore-Tests.

Die 3-2-1-Strategie (3 Kopien, 2 Medientypen, 1 Offsite) ist dabei der Rahmen, in den sich alle Skripte und Cronjobs einordnen sollen.

### 📊 Veranschaulichung – Zielmatrix

| Metrik | Ziel | Umsetzungsidee |
|--------|------|----------------|
| RTO (Recovery Time Objective) | < 4 h | Vollständiger Restore in frische Umgebung inkl. n8n-Smoke-Test |
| RPO (Recovery Point Objective) | < 24 h | Täglicher `pg_dump` + validierte Offsite-Kopie |
| 3 Kopien | Prod + Local + Offsite | DB-Cluster + `/backups` + Remote-Storage |
| 2 Medientypen | Disk + Remote | Droplet-Disk + Cloud-Speicher |
| 1 Offsite | anderer Standort / Provider | z.B. S3/Spaces, verschlüsselt |

---

## 🧱 Backup-Bausteine laut Projektdaten & Architektur

`projektdaten.md` beschreibt PostgreSQL 16-Alpine explizit als **„zentralen Datenpool"** für n8n, Agent-Zero-Queues und E2B-Logs, inkl. täglicher Backups via `pg-backup.sh`, GPG-Verschlüsselung und Remote-Speicher.

`GROUND-ZERO-FINAL-ARCHITEKTUR.md` ergänzt, dass `docker-compose.yml` und `.env` Variablen für Backup-User, Verschlüsselungs-Keys und Netzwerke enthalten und dass Backups fest in den Maschinenraum integriert sind.

### 📊 Veranschaulichung – Datenebenen

| Ebene | Was wird gesichert? | Beispiel |
|-------|---------------------|----------|
| Datenbank | PostgreSQL-Cluster (`postgres_data` Volume) | Tenants, n8n-State, Agent-Zero-Queues, E2B-Logs |
| Konfiguration | `docker-compose.yml`, `env.example`, ggf. DR-Playbooks | Compose-Stack, Env-Struktur, Service-Layout |
| Metadaten/Docs | `projektdaten.md`, Final-Architektur, Erkenntnisse | Projekt-DNA, Backup-Strategie, Lessons Learned |

---

## 🧬 Konkreter DB-Teil (Postgres 16-Alpine)

In `projektdaten.md` steht sehr klar: **Image `postgres:16-alpine`**, Volume (z.B. `/opt/ground-zero/pg-data:/var/lib/postgresql/data`), tägliche Backups via `pg-backup.sh` und Queue-Tabellen für Agent-Zero.

Das heißt: Deine Backup-Strategie ist nicht generisch, sondern genau auf diese DB-Version und diesen Datenstand zugeschnitten.

### 💽 Veranschaulichung – DB-Setup-Snippet (Abstraktion)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n_user"]
volumes:
  postgres_data:
    driver: local
```

---

## 🔄 Backup-Flow (täglich)

Inhaltlich ist der daily Flow in deinen Docs so angedacht: ein **geplanter Dump-Lauf**, Verschlüsselung, Transfer an Remote-Storage und eine Validierungsphase.

Auch wenn die exakten Skripte hier nicht 1:1 im Text stehen, ist das Muster klar: `pg_dump` oder `pg_basebackup` auf dem Droplet, Output in `/backups`, danach Verschlüsselung und Kopie.

### 🔁 Veranschaulichung – ASCII-Pipeline „Daily Backup"

```text
[PostgreSQL 16 Cluster]
        |
        | 1) pg-backup.sh (z.B. via cron @ 03:00)
        v
[/backups/pg/DATE.sql.gz]
        |
        | 2) gpg --encrypt --recipient <key>
        v
[/backups/pg/DATE.sql.gz.gpg]
        |
        | 3) rclone/rsync → Offsite
        v
[Remote Storage (S3/Spaces/...)]
```

---

## 🧪 Validierung & Game-Day-Philosophie

Deine Erkenntnis-Datei betont explizit, dass „Backups ohne Restore-Tests wertlos" sind; DR soll als **Game-Day-Übung** regelmäßig trainiert werden, nicht nur auf Papier existieren.

Das bedeutet: Neben täglichen Dumps brauchst du **regelmäßige Restore-Proben**, idealerweise in einer separaten Umgebung (neues Droplet, anderer Port, Staging-Stack).

### ✅ Veranschaulichung – Game-Day-Checkliste (Kurzform)

- Neues Test-Droplet mit Docker provisionieren.
- Aktuellsten verschlüsselten DB-Dump aus Offsite-Speicher holen.
- Dump entschlüsseln (GPG) und in frischen Postgres-Container einspielen.
- n8n + Agent-Zero mit dieser DB starten (angepasste Ports/URLs).
- Smoke-Tests: ein typischer Workflow, ein Agent-Zero-Job, ein Report.
- Dauer messen (RTO), maximalen Datenverlust berechnen (RPO) und mit Zielwerten vergleichen.

---

## 🧾 Rolle von `docker-compose.yml` & `.env` in DR

`GROUND-ZERO-FINAL-ARCHITEKTUR.md` ordnet `docker-compose.yml` und `.env` explizit den Konfigurations- und Recovery-Artefakten zu: Compose definiert Services, Volumes und Health-Checks, `.env`/`env.example` halten die Parameter, mit denen ein neuer Stack wieder hochgezogen werden kann.

Im DR-Kontext heißt das: **ein vollständiger Restore** beinhaltet nicht nur DB-Daten, sondern auch die Compose-Definitionen und Env-Strukturen aus Git, damit deine Services nach einem Ausfall möglichst identisch wiederhergestellt werden.

### 📊 Veranschaulichung – „Was muss ich sichern?"

| Typ | Beispiel | DR-Relevanz |
|-----|----------|------------|
| DB-Dump | `pg-backup-2025-11-19.sql.gz.gpg` | Absolut kritisch – ohne DB kein Zustand |
| Compose | `docker-compose.yml` | Muss zur DB-Version passen (Services, Ports, Volumes) |
| Env-Struktur | `env.example` | Layout der Variablen, Vault-Mapping |
| Projekt-DNA | `projektdaten.md` | Dokumentiert DR-Strategie & Ziele |

---

## 🧷 Backup-/DR-Strategie im Kontext der Gesamtarchitektur

Weil Postgres der zentrale Datenpool für n8n, Agent-Zero-Queues und E2B-Logs ist, betrifft DR nicht nur „Datenbank-Admins", sondern jede Ebene von Ground-Zero.

- Säule A (Desktop) muss Tools haben, um Backups manuell anzustoßen, Status zu prüfen und Game-Days in Gang zu setzen – z.B. per MCP-Tool `backup.start_daily` oder `dr.validate_latest_restore`.
- Säule B (Maschinenraum) muss die Skripte, Cronjobs und Logging-Pfade bereitstellen, damit Backups zuverlässig laufen und beobachtbar sind.
- Säule C (Sanctum) muss sicherstellen, dass Agent-Zero seine Queues nach einem Restore konsistent sieht (z.B. keine doppelten Jobs, saubere State-Übergänge).

### 🔁 Veranschaulichung – DR-Verzahnung der Säulen

```text
Säule A (Desktop/MCP)
   └─> stößt Backup/Restore-Tasks an (MCP-Tools)

Säule B (Droplets/Compose)
   └─> führt Backups & Restores aus (pg-backup.sh, docker compose)

Säule C (Sanctum/Agent-Zero)
   └─> validiert nach Restore die Queue-/Result-Integrität
```

---

## ✅ Fazit zum DR-Block

- Deine DR-Strategie ist **klar als 3-2-1-Plan mit RTO/RPO-Zielen** definiert, zentriert um einen PostgreSQL-16-Cluster als Single Source of Truth.
- Die notwendigen Artefakte – DB-Dumps, Compose-Files, Env-Strukturen, Projekt-DNA – sind in deinen Dateien bereits identifiziert; es geht nun darum, Skripte/Playbooks und Game-Days daran auszurichten.
- Backup/DR ist tief mit MCP/Desktop, Maschinenraum und Sanctum verzahnt; zukünftige KIs sollen dieses Bild kennen, bevor sie an Skripten oder Infra-Änderungen arbeiten.

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige Backup-/DR-Beschreibung |

---

*Lies als nächstes `05-Monitoring-und-Compliance-Baseline.md` für Details zu Logs, Health-Checks und Security.*
