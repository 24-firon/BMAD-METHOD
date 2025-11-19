# 01 – Projekt-DNA: Ground-Zero Agency Infrastructure

**Version:** 1.0  
**Phase:** Übergreifend (1-3)  
**Säulen:** A (Desktop) + B (Maschinenraum) + C (Sanctum)  
**Datum:** 2025-11-19  

---

## 🧬 Worum es bei Ground-Zero wirklich geht

Ground-Zero ist kein Tool-Sammelsurium und kein Experimentierspielplatz – es ist ein **dreischichtiges Betriebssystem für eine Solo-KI-Agentur**, das bewusst so gebaut ist, dass es mit minimaler manueller Arbeit skaliert, auditierbar bleibt und bei Ausfall in wenigen Stunden wiederherstellbar ist.

Die Kernidee stammt aus der Erkenntnis, dass „klassisches" LLM-Prompting (freie Chat-Befehle, viele Tool-Definitionen im Kontext, ständiges Copy-Paste) **exponentiell ineffizient** wird, sobald man mehr als 5-10 Workflows, mehrere Mandanten und echte Compliance-Anforderungen hat. Ground-Zero löst das durch:

1. **MCP-First-Architektur** – Tools sind klar definierte, wiederverwendbare Bausteine mit Schemas, nicht Ad-hoc-Prompts.
2. **PostgreSQL als Single Source of Truth** – Alle relevanten Daten (n8n-State, Agent-Queues, E2B-Logs, später Analytics) landen in einem Cluster.
3. **Drei-Säulen-Modell** – Kommandozentrale (Desktop), Maschinenraum (Droplets) und Sanctum (Agent-Zero) sind klar getrennt, aber logisch verzahnt.
4. **Phasen-Denken** – Phase 1 (lokal/MCP), Phase 2 (Server/n8n), Phase 3 (Enterprise/OpenBao/Prometheus) als bewusste Eskalations-Stufen, nicht als „irgendwann mal"-Wishlist.

Das Projekt heißt „Ground-Zero", weil es bewusst **bei Null anfängt** – mit klaren Regeln, sauberen Grenzen und dem Ziel, dass jede Entscheidung (Technologie, Prozess, Dokumentation) **nachvollziehbar und reproduzierbar** ist.

### 🎯 Veranschaulichung – Die drei Säulen im Überblick

```text
┌─────────────────────────────────────────────────────┐
│   SÄULE A: Kommandozentrale (Desktop)              │
│   - Windows 11 + WSL2 + Docker Desktop             │
│   - Claude Desktop + MCP-Server (Python)           │
│   - E2B-Sandboxes, Skills, Repo-Verwaltung         │
│   Rolle: Planung, Steuerung, lokale Tests          │
└──────────────────┬──────────────────────────────────┘
                   │  steuert
                   v
┌─────────────────────────────────────────────────────┐
│   SÄULE B: Maschinenraum (Droplets + Docker)       │
│   - Postgres 16 (zentraler Datenpool)              │
│   - n8n (Workflow-Orchestrierung)                  │
│   - Backup/DR-Skripte, Monitoring-Basics           │
│   Rolle: Dauerläufer, Datenhaltung, Automation     │
└──────────────────┬──────────────────────────────────┘
                   │  liefert Jobs/Logs
                   v
┌─────────────────────────────────────────────────────┐
│   SÄULE C: Sanctum (Agent-Zero-Zone)               │
│   - Agent-Zero Engine über Queue-Tabellen          │
│   - Optional: lokaler LLM für sensible Analysen    │
│   Rolle: Geschützter Analyse-Kern, Offline-Tasks   │
└─────────────────────────────────────────────────────┘

        Quer dazu: Monitoring, Compliance, DR
```

---

## 🧱 Die Säulen im Detail

### Säule A: Kommandozentrale (Desktop)

**Zweck:** Deine physische Arbeitsumgebung – hier entstehen Konfigurationen, werden Tests gefahren, Skills geschrieben und MCP-Tools orchestriert.

**Bestandteile:**
- **Windows 11** als Host für Editor (VSCode/Cursor), Git, Claude Desktop
- **WSL2 (Ubuntu)** als Linux-Runtime für Docker und Python-MCP-Server
- **Docker Desktop** für lokale Container (Postgres-Tests, E2B-Sandboxes)
- **Python 3.10+** als MCP-Runtime (bewusst Python statt Node/npm für Kern-Tools)
- **Claude Desktop + MCP-Client** als Haupt-UI für Tool-Aufrufe
- **Repos** (`claude-agents`, `ground-zero-infra`) für Architektur, Docs, Skripte

**Wichtigste Dokumente auf Desktop-Ebene:**
- `projektdaten.md` – der „Speicher" für aktuelle Entscheidungen
- `GROUND-ZERO-FINAL-ARCHITEKTUR.md` – Zielbild
- `ERKENNTNISSE_KOMPLETT.md` – Design-Prinzipien aus Sessions
- Skill-Blueprints, automation-helpers.sh, Phase-Docs

**Rolle im System:**
Säule A ist **nicht** der Ort, wo produktive Workflows laufen – das macht Säule B. Stattdessen ist A der Ort, wo du **planst, konfigurierst, testest und debuggst**, bevor Änderungen auf den Droplets landen.

#### 📊 Veranschaulichung – Desktop-Stack-Komponenten

| Ebene | Komponente | Aufgabe | Phase |
|-------|-----------|---------|-------|
| Host | Windows 11 | GUI, Editor, Git, Claude Desktop | 1+ |
| VM | WSL2 (Ubuntu) | Docker + Python-Runtime | 1+ |
| Runtime | Python 3.10+ | MCP-Server, E2B-Clients | 1+ |
| LLM | Claude Desktop | MCP-Tool-Orchestrierung | 1+ |
| Wissen | projektdaten.md, Final-Architektur, Erkenntnisse | Projekt-DNA | 1-3 |

---

### Säule B: Maschinenraum (Droplets + Docker-Stack)

**Zweck:** Die produktive Laufzeitumgebung – hier laufen n8n, Postgres, Agent-Zero-Service, E2B-Sandbox-Container und alle Dauerläufer.

**Bestandteile:**
- **DigitalOcean Droplet(s)** (Ubuntu-Server) mit Docker Engine
- **PostgreSQL 16-Alpine** als zentraler Datenpool (n8n-DB, Agent-Zero-Queues, E2B-Logs, später Analytics)
- **n8n** (Workflow-Engine) für GitHub-Events, Cron-Jobs, GDPR-Flows, Monitoring-Workflows
- **Agent-Zero-Service** (optional in Phase 2, fest in Phase 3) für Queue-basierte Aufgaben
- **E2B-Sandbox-Container** für isolierte Code-Execution
- **Reverse-Proxy** (Caddy/Traefik) für TLS-Termination und Routing
- **Optional:** Redis (Caching), Worker (Hintergrund-Jobs)

**Zentrale Artefakte:**
- `docker-compose.yml` – Definition aller Services
- `.env` / `env.example` – Umgebungsvariablen-Struktur
- `n8n-postgres-stack.yaml` – Infra-Spec für Stack
- Backup-Skripte (`pg-backup.sh`, `pg-restore.sh`)
- Monitoring-Skripte (`system-audit.sh`, Health-Checks)

**Rolle im System:**
Säule B ist der **Dauerläufer** – alles, was 24/7 erreichbar sein muss, läuft hier. Änderungen an B werden immer von A aus geplant und getestet, nie direkt auf dem Live-System editiert.

#### 📊 Veranschaulichung – Maschinenraum-Services

| Service | Image | Rolle | Abhängigkeiten |
|---------|-------|-------|----------------|
| `postgres` | `postgres:16-alpine` | Zentrale DB (n8n, Queues, Logs) | – |
| `n8n` | `n8nio/n8n:latest` | Workflow-Orchestrierung | postgres |
| `agent-zero` | `groundzero/agent-zero:latest` | Queue-basierte Jobs | postgres |
| `e2b-sandbox` | `e2b/sandbox:latest` | Isolierte Code-Execution | – |
| `redis` | `redis:alpine` (optional) | Caching/Queue | – |
| `reverse-proxy` | `caddy:latest` | TLS-Termination, Routing | n8n, agent-zero |

---

### Säule C: Sanctum (Agent-Zero-Zone)

**Zweck:** Der geschützte Analyse-Kern – hier laufen sensible, rechenintensive oder Offline-Tasks, die nicht direkt im Internet hängen.

**Bestandteile:**
- **Agent-Zero Engine** als eigenständiger Service (Container in Säule B, aber logisch getrennt)
- **Queue-Tabellen** in Postgres (`agent_zero_queue`, `agent_zero_results`) als einzige Kommunikationsschnittstelle
- **Optional:** Lokaler LLM (Ollama/ggml-Modelle) für Analysen ohne Cloud-Abfluss
- **Audit-Tabellen** für nachvollziehbare Job-Historie

**Kommunikations-Pattern:**
Sanctum spricht **nie direkt** mit dem Internet oder anderen Services – alle Inputs kommen über Queue-Tabellen, alle Outputs gehen zurück in Result-Tabellen.

**Rolle im System:**
Säule C ist der Ort für:
- Komplexe Log-Analysen
- Security-Bewertungen
- Mehrstufige Compliance-Checks
- Query-Optimierung
- Dinge, die „zu schwer" für n8n-Workflows sind

#### 🔁 Veranschaulichung – Agent-Zero-Queue-Flow

```text
[n8n / Desktop / andere Dienste]
           |
           |  schreibt Job (JSON Payload, Priorität)
           v
   [agent_zero_queue]  (Postgres-Tabelle)
           |
           |  Agent-Zero pollt neue Jobs
           v
      [Agent-Zero Engine]
           |
           |  führt Aufgabe aus (z.B. Log-Analyse)
           v
   [agent_zero_results] (Postgres-Tabelle)
           |
           |  n8n/Clients lesen Ergebnis
           v
      [Reports / weitere Flows]
```

---

## 🚦 Die drei Phasen als Eskalations-Stufen

Ground-Zero ist nicht „alles auf einmal", sondern bewusst in **drei Ausbaustufen** gedacht, die jeweils neue Funktionen und Komplexität einführen.

### Phase 1: MCP-First (lokal, kein produktiver Server)

**Status:** Aktuell  
**Fokus:** Desktop-Stack, MCP-Server, E2B-Tests, Skills-Setup  
**Säulen aktiv:** Nur A (Desktop)  

**Was läuft:**
- Claude Desktop + Python-MCP-Server (STDIO-Transport)
- Lokale Docker-Container für Tests (Postgres-Instanzen, E2B-Sandboxes)
- Skills (Droplet-Diagnostics, n8n-Health, Repo-Maintenance, Docker-Cleanup) in `.claude/skills`
- Repo-Verwaltung (`claude-agents`, `ground-zero-infra`)

**Was NICHT läuft:**
- Kein produktiver Droplet-Stack
- Kein n8n in Produktion
- Keine Agent-Zero-Engine
- Keine produktiven Backups/DR

**Ziel von Phase 1:**
Einen **stabilen, wiederholbaren Desktop-Setup** haben, der:
- MCP-Tools sauber registriert und aufrufbar macht
- E2B-Sandboxes starten und steuern kann
- Skills und Skripte testet, ohne Live-Systeme zu gefährden

**Typische Aktivitäten:**
- `automation-helpers.sh` entwickeln und testen
- MCP-Manifest (`groundzero-mcp-manifest.json`) entwerfen
- Skill-Blueprints ausarbeiten
- Projektdaten/Final-Architektur pflegen

---

### Phase 2: Server-Stack (n8n + Postgres + Monitoring-Basics)

**Status:** Geplant (nächste Stufe)  
**Fokus:** Droplet-Rebuild, docker-compose-Stack, n8n-Workflows, Backup/DR, Monitoring  
**Säulen aktiv:** A (Desktop) + B (Maschinenraum)  

**Was neu hinzukommt:**
- **Produktiver Droplet** mit `docker-compose.yml`-Stack (Postgres, n8n, Caddy, optional Redis)
- **n8n-Workflow-Library** (8-12 Standard-Workflows: GitHub-Automation, Health-Checks, Backup-Trigger, GDPR-Flows)
- **Backup/DR-Skripte** (`pg-backup.sh`, `pg-restore.sh`, Validation-Script) mit 3-2-1-Strategie
- **Monitoring-Basics** (Logs, Healthchecks, Alerts, wöchentliches `system-audit.sh`)
- **Security-Baseline** (SSH-Keys, UFW, Fail2ban, Auto-Updates)

**Was noch NICHT läuft:**
- Kein OpenBao (Secrets noch in `.env`)
- Kein Prometheus/Grafana (nur Shell-Checks + Cron)
- Agent-Zero optional als Proof-of-Concept, nicht als Voll-Setup

**Ziel von Phase 2:**
Einen **produktiven, aber minimal komplexen Server-Stack** haben, der:
- RTO < 4 h, RPO < 24 h erfüllt
- 90% Compliance-Threshold schafft (wöchentliches Audit-Script)
- n8n-Workflows für GitHub, Monitoring und GDPR orchestriert
- Vom Desktop aus steuerbar ist (MCP-Tools + Skills)

**Typische Aktivitäten:**
- Droplet provisionieren, `docker-compose up -d`
- n8n-Workflows importieren, testen, dokumentieren
- Backup-Job in Cron einrichten, ersten Game-Day fahren
- Monitoring-Alerts (Email + optional Webhook) konfigurieren

---

### Phase 3: Enterprise-Level (OpenBao, Prometheus, Voll-Sanctum)

**Status:** Vision (längerfristig)  
**Fokus:** Secrets-Management, vollwertige Observability, Agent-Zero produktiv, Analytics  
**Säulen aktiv:** A + B + C (alle)  

**Was neu hinzukommt:**
- **OpenBao** als Vault-Ersatz für Secrets (statt `.env`)
- **Prometheus + Grafana** für Metriken, Dashboards, SLO-basierte Alerts
- **Loki / ELK** für zentrale Log-Aggregation
- **Agent-Zero Engine** als produktiver Service mit Queue-Tabellen
- **Optional:** Lokaler LLM (Ollama) in Sanctum
- **Multi-Tenant-Support** via PostgreSQL RLS
- **SLA-Monitoring** mit Breach-Notifier, Reports

**Ziel von Phase 3:**
Ein **Enterprise-taugliches Setup**, das:
- Secrets rotiert und auditiert (OpenBao)
- SLOs definiert und überwacht (Prometheus/Grafana)
- Komplexe Analysen in Sanctum auslagert (Agent-Zero)
- Skaliert auf 1000+ Tasks/Woche

**Trigger für Phase 3:**
- Phase 2 läuft stabil (mehrere Monate)
- Backup/DR mehrfach getestet
- Mandanten-Daten wachsen (Multi-Tenant wird relevant)
- OpenBao-Migration geplant (Secrets-Rotation nötig)

---

## 🧠 Kern-Prinzipien (die "DNA")

### 1. Editor statt Chatbot

**Was es bedeutet:**  
Prompts, Playbooks, Skripte und Konfigs sind **Artefakte wie Infrastructure as Code** – sie werden in Git versioniert, in Markdown/YAML/Python geschrieben und systematisch gepflegt, nicht als Wegwerf-Chat-Text behandelt.

**Praktisch:**
- Skill-Definitionen liegen in `.claude/skills/*.md`
- MCP-Tools haben JSON-Schemas in einem Manifest
- Workflows sind in n8n dokumentiert oder als YAML-Snippets exportiert
- Projektdaten/Final-Architektur sind die „Quelle der Wahrheit", nicht Chat-Historie

**Warum:**
Weil „Frage im Chat → neue Antwort" **nicht skaliert**, sobald du mehr als 5 Workflows hast oder nach einer Pause zurückkommst. Mit versionierten Artefakten kannst du jederzeit nachvollziehen, was du wann entschieden hast.

---

### 2. MCP-First & Hybrid-Execution

**Was es bedeutet:**  
Tools sind **klar definierte, wiederverwendbare Bausteine** (MCP-Tools mit Schemas), nicht Ad-hoc-Shell-Befehle. Code-Execution läuft **isoliert** (E2B-Sandboxes), nicht direkt auf deinem Host oder Droplet.

**Praktisch:**
- Claude Desktop ruft `tools/http.get`, `tools/fs.read`, `tools/e2b.run` auf – nicht „führe diesen Bash-Befehl aus"
- E2B-Sandboxes kapseln riskante/schwere Operationen (Magic-MCP-Server, große Scripts)
- Cloud-Code (GitHub Actions) für wiederkehrende, skalierbare Tasks

**Warum:**
Der MCP-Report zeigt: Klassische Tool-Nutzung (alle Definitionen + Zwischenergebnisse im Prompt) explodiert bei 20+ Tools. Mit MCP/E2B sparst du **98,7% Token** und hast gleichzeitig bessere Auditierbarkeit.

---

### 3. PostgreSQL als Single Source of Truth

**Was es bedeutet:**  
Alle relevanten Daten landen in **einem PostgreSQL-16-Cluster** – n8n-State, Agent-Zero-Queues, E2B-Logs, später Analytics, GDPR-Audit-Logs, SLA-Metriken.

**Praktisch:**
- n8n nutzt Postgres statt SQLite
- Agent-Zero schreibt Jobs in `agent_zero_queue`, Results in `agent_zero_results`
- Backups sichern **nur** diesen einen Cluster (plus `.env`/Compose als Konfig)
- Kein zweiter MySQL, kein MongoDB, keine verteilten Datenbanken

**Warum:**
Weil **ein Restore-Punkt** viel einfacher ist als „5 Datenbanken + 3 File-Systeme + 2 Caches". RTO < 4 h ist realistisch, wenn du nur Postgres + Compose wiederherstellen musst.

---

### 4. Drei-Säulen-Trennung (Separation of Concerns)

**Was es bedeutet:**  
Desktop (A) steuert, Maschinenraum (B) führt aus, Sanctum (C) analysiert – **niemals vermischen**.

**Praktisch:**
- Du editierst nie direkt auf dem Droplet (kein `vim docker-compose.yml` auf dem Server)
- Agent-Zero hat keinen direkten Internet-Zugang, nur Queue-Tabellen
- MCP-Tools auf Desktop können Droplet-Skripte **anstoßen**, aber nicht unkontrolliert Shell-Befehle ausführen

**Warum:**
Weil **klare Grenzen** Fehler reduzieren und Audits ermöglichen. Wenn jeder Layer seinen Scope hat, kannst du schnell erkennen, wo ein Problem liegt.

---

### 5. Phasen-Denken (keine Big-Bang-Migration)

**Was es bedeutet:**  
Jede Phase baut auf der vorherigen auf, aber **führt neue Komplexität nur ein, wenn die alte stabil ist**.

**Praktisch:**
- Phase 1: MCP lokal testen, bevor du Droplets anfasst
- Phase 2: n8n + Monitoring-Basics laufen lassen, bevor Prometheus kommt
- Phase 3: OpenBao erst, wenn Secrets-Rotation wirklich nötig wird

**Warum:**
Weil „alles auf einmal" in 99% der Fälle scheitert. Mit klaren Phasen kannst du jederzeit pausieren und hast trotzdem ein funktionierendes System.

---

### 6. Compliance & DR als First-Class-Themen

**Was es bedeutet:**  
Backup, Monitoring, Security und GDPR sind **keine Add-Ons**, sondern von Anfang an eingeplant.

**Praktisch:**
- RTO < 4 h, RPO < 24 h als harte Ziele in Phase 2
- Wöchentliches `system-audit.sh` mit 90% Pass-Threshold
- GDPR-Flows (Auskunft, Löschung) als n8n-Workflows in Phase 2
- 3-2-1-Backup (3 Kopien, 2 Medien, 1 Offsite) von Tag 1

**Warum:**
Weil „später kümmern wir uns um Backups" in der Realität nie passiert. Mit klaren RTO/RPO-Zielen und wöchentlichen Audits **merkst du sofort**, wenn etwas kaputt geht.

---

## ⚠️ Harte Regeln (Non-Negotiable)

Diese Regeln gelten **immer**, in allen Phasen, für alle Agents:

1. **Keine Shell-Execs durch KI ohne Freigabe** – KIs dürfen Skripte vorschlagen, aber nicht eigenmächtig auf Produktivsystemen ausführen.
2. **Keine Browser-Storage-APIs in Artifacts** – `localStorage`, `sessionStorage`, `document.cookie` werfen SecurityError in Sandboxes.
3. **Keine Secrets in Git/Logs** – `.env` ist gitignored, Secrets werden nie geloggt.
4. **Backup vor jedem großen Change** – Vor Postgres-Migrationen, Compose-Änderungen, Server-Rebuilds immer Backup.
5. **Wöchentliches Audit-Script muss laufen** – Wenn `system-audit.sh` < 90% Pass hat, **sofort reagieren**.
6. **MCP-Tools nur mit Whitelists** – Erlaubte Pfade/Kommandos explizit definiert, alles andere Tabu.
7. **Änderungen an Säule B nur von Säule A aus** – Nie direkt auf Droplet editieren.
8. **Skills/Agents müssen Guardrails haben** – Jeder Skill definiert erlaubte/verbotene Pfade und Kommandos.

---

## 🔗 Wie alles zusammenhängt

### Datenfluss (vereinfacht)

```text
[Du / Editor]
      |
      v
[Claude Desktop + MCP-Client]
      |
      v
[MCP-Server (Python)]
      |
      +--> [Filesystem] (projektdaten.md, Skripte, Skills)
      |
      +--> [E2B-Sandbox] (isolierte Code-Exec)
      |
      +--> [HTTP-APIs] (GitHub, DO, n8n-Health)
      |
      v
[Droplet: docker-compose-Stack]
      |
      +--> [Postgres] <-- [n8n] <-- [GitHub-Webhooks]
      |
      +--> [Agent-Zero] <-- [Queue-Tabellen]
      |
      +--> [Backup-Skripte] --> [Offsite-Storage]
      |
      v
[Monitoring/Alerts] --> [Email / Webhook]
```

### Workflow-Beispiel: GitHub Issue → n8n → Postgres → Agent-Zero

1. **GitHub Issue erstellt** → Webhook an n8n
2. **n8n-Workflow** "GitHub Issue → Postgres Log"  
   - Extrahiert Issue-Daten (Titel, Body, Labels)
   - Schreibt in `github_events`-Tabelle (Postgres)
3. **Optional:** n8n schreibt Job in `agent_zero_queue`  
   - Payload: `{ "type": "analyze_issue", "issue_id": 123 }`
4. **Agent-Zero** pollt Queue, sieht neuen Job  
   - Lädt Issue-Daten aus Postgres
   - Führt Analyse aus (z.B. Sentiment, Kategorisierung)
   - Schreibt Ergebnis in `agent_zero_results`
5. **n8n-Workflow** "Agent-Zero Result → GitHub Comment"  
   - Liest Result aus Postgres
   - Postet Analyse als GitHub-Comment

---

## 📊 Wichtigste Metriken & Ziele

| Metrik | Ziel | Phase | Verifizierung |
|--------|------|-------|---------------|
| RTO (Recovery Time Objective) | < 4 h | 2 | Game-Day-Tests, DR-Runbook |
| RPO (Recovery Point Objective) | < 24 h | 2 | Tägliche Backups, Validierung |
| Compliance-Pass-Rate | ≥ 90% | 2 | Wöchentlich `system-audit.sh` |
| Backup-Frequenz | Täglich | 2 | Cron + Monitoring-Alert |
| MCP-Tool-Registrierung | 6-10 Tools | 1 | Manifest + Desktop-Test |
| n8n-Workflow-Library | 8-12 Workflows | 2 | Workflow-Doku + Tests |
| Skills aktiv | 4-6 Skills | 1-2 | `.claude/skills/` + Tests |

---

## 🚀 Nächste Schritte (Phase 1 → Phase 2)

### Phase 1 abschließen (aktuell)
- [ ] Python-MCP-Server mit 6+ Tools produktiv
- [ ] E2B-Sandbox-Tests erfolgreich
- [ ] Skills (Droplet-Diagnostics, n8n-Health, Repo-Maintenance, Docker-Cleanup) getestet
- [ ] `automation-helpers.sh` funktional
- [ ] Projektdaten/Final-Architektur aktuell

### Phase 2 vorbereiten
- [ ] Droplet-Analyse (bestehende Server, VPN, SSH-Keys)
- [ ] `docker-compose.yml` final (Postgres, n8n, Caddy, optional Redis)
- [ ] `env.example` mit allen Keys
- [ ] Backup-Skripte (`pg-backup.sh`, `pg-restore.sh`) geschrieben
- [ ] Monitoring-Skripte (`system-audit.sh`, Health-Checks) vorbereitet

### Phase 2 starten
- [ ] Droplet provisionieren (Ubuntu, Docker, UFW, Fail2ban)
- [ ] `docker-compose up -d`, Health-Checks verifizieren
- [ ] n8n-Workflow-Library importieren
- [ ] Backup-Cron einrichten, ersten Backup-Test fahren
- [ ] Ersten Game-Day durchführen (RTO/RPO messen)

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige DNA-Beschreibung aus allen Quellen synthetisiert |

---

*Dieses Kapitel bildet die Grundlage für alle weiteren Dokumente. Lies es zuerst, bevor du in Details einsteigst.*
