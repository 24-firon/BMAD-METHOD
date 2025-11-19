# 03 – Maschinenraum & Docker-Compose-Stack

**Version:** 1.0  
**Phase:** 2 (Server-Stack produktiv)  
**Säule:** B (Maschinenraum)  
**Datum:** 2025-11-19  

---

## 🏭 Maschinenraum-Zielbild (Droplets + Docker-Stack)

Der Maschinenraum ist Säule B: ein oder mehrere Ubuntu-Droplets mit Docker, auf denen alle Kernservices als Compose-Stack laufen.

Ziel: Ein **klar definierter `docker-compose.yml`-Stack**, der n8n, Postgres, Agent-Zero, E2B-Sandbox und optional Redis/Proxy/Worker startet – reproduzierbar, versioniert und mit dokumentierten Health-Checks.

### 🔁 ASCII-Bild – Maschinenraum im Überblick

```text
+-----------------------------------------------+
| Droplet (Ubuntu)                              |
|  Docker Engine                                |
|   ├─ postgres (DB 16-alpine)                  |
|   ├─ n8n (Workflows)                          |
|   ├─ agent-zero (Sanctum-Gateway)            |
|   ├─ e2b-sandbox (Code-Execution)            |
|   ├─ redis (optional)                         |
|   └─ reverse-proxy (z.B. Caddy/Traefik)      |
+-----------------------------------------------+
        │
        └─> VPN / HTTPS / SSH für externen Zugriff
```

---

## 🧩 Services im Compose-Stack

Aus `GROUND-ZERO-FINAL-ARCHITEKTUR.md`: die Services, die im Maschinenraum laufen sollen.

| Service | Rolle | Wichtigste Aufgaben |
|---------|------|---------------------|
| `postgres` | Zentraler Datenpool | n8n-DB, Agent-Zero-Queues, E2B-Logs, spätere Analytics |
| `n8n` | Orchestrierung | GitHub-Events, Cron-Jobs, Compliance-Workflows, DR-Runs |
| `agent-zero` | Sanctum-Gateway | Arbeitet auf Queue-Tabellen, führt Analysen/Tasks aus |
| `e2b-sandbox` | Code-Execution | Isolierte Ausführung von Skripten/Tools |
| `redis` (optional) | Cache/Queue | Zwischenpuffer, schnelle Queues, Rate-Limit-Shields |
| `reverse-proxy` | Zugriff & TLS | Routing, HTTPS-Terminator, evtl. Auth |
| `worker` (optional) | Hintergrund-Jobs | Batch-/Cleanup-Tasks, Reports |

---

## 🌐 Netzwerk & Ports

Final-Architektur beschreibt ein eigenes Docker-Netzwerk und dedizierte Ports nach außen.

### 📊 Beispiel-Port-Matrix

| Dienst | Interner Port | Externer Port | Zugriffsweg |
|--------|---------------|---------------|-------------|
| `n8n` | `5678` | `5678` oder via Proxy | VPN/HTTPS only |
| `postgres` | `5432` | – (nur intern im Netzwerk) | Nur Docker-Netzwerk `n8n-network` |
| `agent-zero` | z.B. `8081` | via Proxy | Intern/Queue-basiert |
| `e2b-sandbox` | je nach Image | nur intern | von MCP/E2B-Client genutzt |
| `reverse-proxy` | `80` / `443` | `80` / `443` | Public/TLS-Terminator |

### 🔁 ASCII-Netzwerk-Skizze

```text
               Internet / VPN
                 |
          +------+------+
          |  reverse-   |
          |   proxy     |
          +------+------+
                 |
        +--------+--------+
        |   n8n-network   |
        +---+------+------+
            |      |
       +----+--+  +--------+
       | n8n  |  | postgres|
       +------+  +--------+
            |
       +----+----------+
       | agent-zero    |
       +---------------+
            |
       +----+----------+
       | e2b-sandbox   |
       +---------------+
```

---

## 💾 Volumes & Datenhaltung

Aus Final-Architektur + Projektdaten:

| Volume | Pfad im Host (Beispiel) | Mount im Container | Inhalt |
|--------|-------------------------|--------------------|--------|
| `postgres_data` | `/opt/ground-zero/pg-data` | `/var/lib/postgresql/data` | DB-Daten (PostgreSQL 16-Cluster) |
| `n8n_data` | `/opt/ground-zero/n8n-data` | `/home/node/.n8n` | Workflows, Credentials, n8n-State |
| `agent_zero_data` (optional) | `/opt/ground-zero/agent-zero` | je nach Image | Config/Cache von Agent-Zero |
| `e2b_data` (optional) | `/opt/ground-zero/e2b` | je nach Image | Cache für Images/Logs |

Diese Volumes hängen direkt an deiner Backup-Strategie (tägliche Dumps/Syncs, GPG-Verschlüsselung, Remote-Speicher).

---

## ⚙️ `docker-compose.yml` – Rolle und Regeln

Final-Architektur beschreibt `docker-compose.yml` als zentrales Dokument für den Maschinenraum.

### 📋 Kernregeln für `docker-compose.yml`

- Enthält **alle Services**, die im Maschinenraum laufen sollen (siehe Service-Tabelle).
- Ist **Git-versioniert**; Prod-Releases werden via Tags markiert, damit man genau weiß, mit welcher Compose-Version ein bestimmter Zustand gefahren wurde.
- Nutzt ein eigenes Netzwerk (z.B. `n8n-network`) für interne Kommunikation.
- Setzt Health-Checks für kritische Dienste wie Postgres und n8n (z.B. `pg_isready` und HTTP-Health-Endpoints).

Ein abstrahiertes Compose-Snippet (Struktur, nicht 1:1-Code):

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n_user"]
  n8n:
    image: n8nio/n8n:latest
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n_user
  agent-zero:
    image: groundzero/agent-zero:latest
  e2b-sandbox:
    image: e2b/sandbox:latest

volumes:
  postgres_data:
  n8n_data:
```

---

## 🔐 `.env` und `env.example`

Final-Architektur gibt ein klares Muster für `.env`/`env.example` vor.

### 📋 Beispiel-Inhalt `env.example`

```text
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
POSTGRES_USER=n8n
POSTGRES_PASSWORD=[generated-random]
POSTGRES_DB=n8n
N8N_ENCRYPTION_KEY=[generated-random]
BACKUP_ENCRYPTION_KEY=[generated-random]
```

**Regeln**:

- `env.example` ist **dokumentarisch** und liegt im Repo, aber enthält NIE echte Secrets – nur Platzhalter/Struktur.
- `.env` oder `.env.production` liegen **nur auf dem Droplet** (oder in einem Secrets-Store) und sind via `.gitignore` ausgeschlossen.
- Langfristig ersetzt ein Vault (OpenBao) `.env` für produktive Secrets, `.env` wird dann nur noch für lokale Dev-Runs verwendet.

---

## 📈 Health-Checks & Monitoring im Maschinenraum

Auch wenn Observability-Details in eigenen Phase-Docs kommen, die DB/Service-Health ist bereits jetzt konzeptionell verankert.

### 📊 Health-Check-Bausteine

| Ebene | Mechanismus | Beispiel |
|-------|-------------|----------|
| DB | `pg_isready` | In Compose-Healthcheck für Postgres |
| n8n | HTTP-Endpoint | GET `/health` / `/rest/health` |
| Agent-Zero | Custom-Ping | z.B. `/healthz` oder Queue-Self-Check |
| E2B | Sandbox API | Test-Run + Codes/Timeouts |

ASCII-Mini-Flow (n8n-Start abhängig von Postgres):

```text
[postgres] --healthy?--> yes --> [n8n startet]
                         no  --> [warten / restart policy]
```

---

## 🚀 Cold-Start & Redeploy – Maschinenraum-Playbook (Kurzfassung)

Für Phase 2 wird ein wiederholbares Playbook gebraucht, das sich auf diese Infrastruktur stützt.

### 🔁 Cold-Start (neues Droplet)

1. **Droplet provisionieren** (Ubuntu, Docker, SSH-Keys, UFW-Regeln).
2. Repo `ground-zero-infra` klonen, `docker-compose.yml` und `env.example` auf das Droplet bringen.
3. `.env` aus Secrets-Store/1Password/OpenBao erzeugen.
4. `docker compose up -d` starten, Health-Checks beobachten (Postgres, n8n, Agent-Zero, E2B).
5. Erste n8n-Login, Konfiguration prüfen, ggf. Workflows importieren.

### 🔁 Redeploy / Update

1. Git-Pull auf Droplet, neue `docker-compose.yml`/Images holen.
2. `docker compose pull && docker compose up -d` mit `--no-deps` selektiv oder voll.
3. Validierung: Health-Checks, Test-Workflow, ggf. Agent-Zero/E2B-Smoke-Test.

---

## 🎯 Was der Maschinenraum-Block jetzt leistet

- Zeigt, **welche Services** du im Maschinenraum erwartest, mit Rollen und Abhängigkeiten.
- Macht **Netzwerk, Ports und Volumes** sichtbar, statt sie im Text zu verstecken.
- Verknüpft `.env`/`env.example` sauber mit dem Zielbild und der späteren Vault-Strategie.
- Liefert ein **Cold-Start/Redeploy-Gerüst**, an dem sich sowohl du als auch spätere Agents orientieren können.

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige Maschinenraum-Beschreibung |

---

*Lies als nächstes `04-Backup-und-DR-Stack.md` für Details zu Backups und Disaster Recovery.*
