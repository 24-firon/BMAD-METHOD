# 09 – groundzero-MCP-Server: Das zentrale Werkzeug

**Version:** 1.0  
**Phase:** 1-2 (Design in 1, produktiv in 2)  
**Säule:** A (Desktop), wirkt auf B  
**Datum:** 2025-11-19  

---

## 🧠 MCP-Server „groundzero" – Design im Kontext

In deinen Launch-Dokumenten ist für Instanz 8 ein kompletter Design-Auftrag für einen „groundzero-MCP-Server" formuliert, inklusive Architektur-Dokument, Manifest, Tool-Liste, Sicherheitsgrenzen und Desktop-Integration.

Der MCP-Server soll als zentrale, streng begrenzte Schnittstelle agieren, die lokale Skripte (`scripts/analysis`, `scripts/backup`, `scripts/desktop`), Spec-Dateien (`infra`, `spec/infra`, `docs`) und ggf. externe APIs (DigitalOcean, GitHub) erreichbar macht, ohne direkte Shell-Freiheit.

Der Transport ist zweistufig gedacht: in Phase 1 ein lokaler STDIO-Server auf WSL2 für Desktop-Integration, später optional eine HTTP/SSE-Variante, z.B. hinter einem API-Gateway für Remotenutzung.

### 🔁 Veranschaulichung – MCP-Server im System

```text
[Claude Desktop]
      |
      | (MCP-Client: stdio/http)
      v
[groundzero-MCP-Server]
      |
      +--> scripts/analysis (droplet-full-diagnostics.sh, n8n-healthcheck)
      |
      +--> scripts/backup (pg-backup.sh, pg-restore.sh, backup-validation.sh)
      |
      +--> spec/infra/*.yaml (n8n-postgres-stack, monitoring-basics)
      |
      +--> optional: APIs (GitHub, DO) via klaren Tools
```

---

## 🧩 MCP-Tool-Klassen und Manifest-Ideen

Das Instanz-8-Prompt definiert explizit mehrere Tool-Klassen, die im Manifest auftauchen sollen, z.B. Read-Only-Infra-Tools, Backup/DR-Tools und Spec-Inspector-Tools.

Jedes Tool soll Name, Beschreibung, Input-Schema, Output-Schema und Sicherheits-Hinweise erhalten, etwa „nur lesend", „nur Dry-Run in Phase 1", „keine destruktiven Docker-Kommandos".

Beispiele aus dem Prompt: `listspecs` (Spec-Dateien auflisten), `validateinfrastack` (Konsistenz von n8n-Stack-Spec und Compose prüfen), `rundropletdiagnostics` (D1-Diagnose planen), `runn8nhealthcheck`, `listbackups` und `backupstatussummary`.

### 📊 Veranschaulichung – Beispiel-Tool-Katalog (konzeptionell)

| Tool | Klasse | Aufgabe | Sicherheits-Note |
|------|--------|--------|------------------|
| `listspecs` | Spec-Inspector | Listet alle `spec/infra/*.yaml` mit Kurzbeschreibung | rein lesend |
| `validateinfrastack` | Spec-Inspector | Vergleicht `spec/infra/n8n-postgres-stack.yaml` mit `infra/docker-compose.sample.yml` | keine Änderungen, Report only |
| `rundropletdiagnostics` | Infra-Tool | Führt Droplet-Diagnose-Script indirekt aus bzw. plant dies | Phase 1 nur lokal/Dry-Run |
| `runn8nhealthcheck` | Infra-Tool | Läuft `n8n-postgres-healthcheck.sh` und fasst Resultat zusammen | nur lesend |
| `listbackups` | Backup-Tool | Listet vorhandene Backups in `/backups` | nur lesend |
| `backupstatussummary` | Backup-Tool | Aggregiert Output von `backup-validation.sh` | nur lesend |

---

## 🏗️ MCP-Server-Skeleton und Desktop-Integration

Das Instanz-8-Prompt sieht ein Architektur-Dokument `docs/mcp/groundzero-mcp-architecture.md`, ein Manifest `docs/mcp/groundzero-mcp-manifest.json` und ein `groundzero-mcp-server-skeleton` vor, der nur Beispiel-Code liefert.

Der Server-Skeleton soll das Manifest laden, Tools registrieren und einen STDIO-Transport für Claude Desktop bereitstellen, ohne bereits produktiven Code auszurollen; konkrete Implementierungen bleiben Pseudocode mit Kommentaren zu Pfaden, Validierung und Fehlerbehandlung.

Zusätzlich soll `docs/desktop/groundzero-mcp-integration.md` erklären, wie du den Server lokal startest, in Desktop registrierst und mit bestehenden Skills kombinierst, z.B. Droplet-Diagnostics-Skill plus `backupstatussummary`.

### 📋 Veranschaulichung – minimaler Projekt-Skeleton (Struktur)

```text
groundzero-mcp-server/
  README.md
  manifest/groundzero-mcp-manifest.json
  src/
    server.py          # lädt Manifest, registriert Tools, startet stdio
    tools_specs.py     # listspecs, validateinfrastack (nur lesend)
    tools_backup.py    # listbackups, backupstatussummary (nur lesend)
    tools_diagnostics.py  # rundropletdiagnostics, runn8nhealthcheck (Dry-Run)
  .env.example         # MCP_PORT, LOG_LEVEL, optional Tokens
```

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige MCP-Server-Design-Beschreibung |

---

*Damit ist das komplette Dokumentations-Set fertig. Lies `00-INDEX.md` für die Übersicht.*
