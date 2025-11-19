# BMAD DEVELOPER TEAM - PLANUNGSAUFTRAG
# Ground-Zero Multi-Agent Orchestration Infrastructure

## 📋 MISSION

Ihr seid ein Multi-Agent Development Team. Erstellt einen **detaillierten, abgestimmten Implementierungsplan** für:
1. Behebung von 52 kritischen Fehlern (5 Blocker)
2. Deployment der Ground-Zero Infrastructure (Phase 1.2)
3. Production-Ready Setup mit Tests & Monitoring

---

## 🏗️ WAS IST GROUND-ZERO? (Projekt-Kontext)

### Die 3-Pillar-Architektur

**Säule 1: Kommandozentrale (Windows 11)**
- Claude Desktop mit MCP-Integration
- Git-Client, Obsidian (Doku)
- Lokale Development-Tools

**Säule 2: Maschinenraum (Ubuntu 24.04 / Docker)**
- **n8n** (Workflow-Orchestrator) - 5 Production Workflows
- **PostgreSQL 16** (State + Checkpoints) - Source of Truth
- **Redis 7** (Queue + Cache) - Priority Queue mit ZPOPMIN
- **Ollama** (LLM-Host) - phi3:mini Model für Reasoning
- **Agent Zero** (KI-Agent) - FastAPI + Queue Processor

**Säule 3: Sanctum (Agent Zero)**
- Autonomer Task-Processor
- Queue-basierte Verarbeitung
- State-Management mit Checkpoints
- LLM-Integration via Ollama

---

## 🔄 DATENFLUSS (End-to-End)

```
1. User → Claude Desktop (Kommandozentrale)
2. Claude → groundzero-MCP-Server (11 Tools)
3. MCP → n8n Orchestrator Workflow (HTTP POST)
4. Orchestrator → PostgreSQL (Checkpoint erstellen)
5. Orchestrator → n8n Worker (Task delegieren)

   [EINFACH]
   6a. Worker → Direkte Verarbeitung
   7a. Worker → n8n Critic (Validierung)

   [KOMPLEX]
   6b. Worker → Agent Zero (Queue POST)
   7b. Agent Zero → Ollama LLM (Reasoning)
   8b. Agent Zero → PostgreSQL (Result)
   9b. Result → n8n Critic

10. Critic → PostgreSQL (Final Result speichern)
11. PostgreSQL → Claude Desktop (via MCP notification)
```

### Technische Details:
- **State-Sync**: PostgreSQL = Source of Truth, Redis = Cache (1h TTL)
- **Queue**: Redis Sorted Set (ZADD priority, BZPOPMIN consume)
- **Checkpoints**: Bei jedem Schritt in PostgreSQL gespeichert
- **Error-Recovery**: Retry 3x, dann Dead-Letter-Queue
- **Monitoring**: Prometheus Metrics + Structured Logging

---

## 🎯 TECHNOLOGIE-STACK (Vollständig)

### Backend Services:
- **Python 3.11** (Agent Zero, MCP-Server)
  - FastAPI 0.104.1 (REST API)
  - asyncpg 0.29.0 (PostgreSQL async)
  - redis 5.0.1 (Redis async client)
  - ollama 0.1.6 (LLM client)
  - structlog 23.2.0 (Structured Logging)
  - prometheus-client 0.19.0 (Metrics)

- **Node.js** (n8n)
  - n8n 1.33.0 (Workflow Engine)
  - PostgreSQL Driver
  - Redis Bull Queue

### Infrastructure:
- **Docker Compose 3.8** (5 Services orchestriert)
- **PostgreSQL 16-alpine** (2GB RAM, alpine für Security)
- **Redis 7-alpine** (512MB RAM, LRU eviction)
- **Ollama latest** (4GB RAM, CPU-based)

### Integration:
- **MCP (Model Context Protocol)** - Claude Desktop Integration
  - Filesystem MCP (npx)
  - GitHub MCP (npx)
  - groundzero MCP (Python STDIO)

---

## ⚠️ AKTUELLER STATUS: 52 FEHLER GEFUNDEN

### Fehlerverteilung nach Komponente:

| Komponente | Kritisch | Hoch | Mittel | Gesamt |
|------------|----------|------|--------|--------|
| Agent Zero Service | 3 | 3 | 7 | 13 |
| groundzero-MCP-Server | 4 | 3 | 4 | 11 |
| docker-compose.yml | 3 | 5 | 6 | 14 |
| n8n Workflows (5 Files) | 5 | 4 | 3 | 12 |
| Shell Scripts (3 Files) | 4 | 5 | 3 | 12 |
| Config Files (.env, MCP) | 6 | 1 | 2 | 9 |
| **TOTAL** | **25** | **21** | **25** | **71** |

---

## 🔴 TOP 5 BLOCKER-ISSUES (Ohne Fix: System startet NICHT)

### 1. SQL INDEX-Syntax-Fehler in state_manager.py
**Datei**: `services/agent-zero/state_manager.py` (Zeilen 71-72, 88-89)
**Problem**: MySQL-Syntax `INDEX idx_name (column)` in CREATE TABLE
**PostgreSQL erwartet**: Separate `CREATE INDEX` Statements
**Impact**: CREATE TABLE schlägt fehl → Agent Zero startet nicht
**Fix-Aufwand**: 5 Minuten

```python
# FALSCH (MySQL):
CREATE TABLE agent_zero_checkpoints (
    id SERIAL PRIMARY KEY,
    task_id VARCHAR(255) NOT NULL,
    INDEX idx_task_id (task_id)  # ❌ PostgreSQL unterstützt das NICHT
);

# RICHTIG (PostgreSQL):
CREATE TABLE agent_zero_checkpoints (
    id SERIAL PRIMARY KEY,
    task_id VARCHAR(255) NOT NULL
);
CREATE INDEX idx_task_id ON agent_zero_checkpoints (task_id);
```

---

### 2. Missing Imports in MCP-Server Tools
**Dateien**:
- `groundzero-mcp-server/tools/health_tools.py` (Zeilen 1-5)
- `groundzero-mcp-server/tools/system_tools.py` (Zeilen 1-4)

**Fehlende Imports**:
```python
import os                    # ❌ FEHLT - verwendet in Line 27, 28
from datetime import datetime # ❌ FEHLT - verwendet in Line 58, 71
```

**Impact**: NameError beim MCP-Server-Start
**Fix-Aufwand**: 2 Minuten

---

### 3. Falsche ENV-URLs in .env.example
**Datei**: `.env.example` (Zeilen 38, 50)

**Problem**: `localhost` funktioniert NICHT in Docker-Containern!
```bash
# FALSCH:
REDIS_URL=redis://localhost:6379/1      # ❌ Agent Zero sieht localhost nicht
POSTGRES_URL=postgresql://...@localhost:5432/...  # ❌

# RICHTIG:
REDIS_URL=redis://redis:6379/1          # ✅ Docker Service-Name
POSTGRES_URL=postgresql://...@postgres:5432/...   # ✅
```

**Impact**: Agent Zero kann nicht zu Redis/PostgreSQL verbinden
**Fix-Aufwand**: 1 Minute

---

### 4. Ollama container_name fehlt
**Datei**: `docker-compose.yml` (Zeile 157)

**Problem**: Service hat KEINE `container_name` definiert
```yaml
# AKTUELL:
ollama:
  image: ollama/ollama:latest
  # container_name: ??? FEHLT!

# MCP-Tools erwarten:
docker logs ground-zero-ollama  # ❌ Container heißt anders (auto-generated)
```

**Impact**: MCP-Tool `read_logs("ollama")` schlägt fehl
**Fix-Aufwand**: 1 Minute (eine Zeile hinzufügen)

---

### 5. PostgreSQL Table "emergency_stops" fehlt
**Datei**: `infra/init/init-postgres-schema.sql` (nicht vorhanden)

**Problem**: emergency_tools.py versucht INSERT INTO emergency_stops
```python
# Code erwartet:
await conn.execute("""
    INSERT INTO emergency_stops (timestamp, reason, actions_taken)
    VALUES (NOW(), $1, $2)
""", reason, actions)
```

**Aber**: Table existiert NICHT in init-postgres-schema.sql!

**Impact**: emergency_stop() MCP-Tool crashed mit SQL-Error
**Fix-Aufwand**: 5 Minuten (CREATE TABLE schreiben)

---

## 📊 DETAILLIERTE FEHLER-KATEGORIEN

### Agent Zero Service (13 Fehler)

**KRITISCH (3):**
1. SQL INDEX-Syntax → CREATE TABLE failed
2. Pydantic v2 `.dict()` → Sollte `.model_dump()` sein (Zeilen 168, 246, 261)
3. NULL-Pointer: `datetime.isoformat()` ohne NULL-Check (Zeilen 202-203)

**HOCH (3):**
4. Falsche Duration bei Timeout (Zeile 227-232)
5. Missing queue_manager NULL-Check (Zeile 168)
6. `time.time()` Import inline statt am Top

**MITTEL (7):**
7-13. Deprecated `datetime.utcnow()`, aioredis redundant, ollama Version alt, etc.

---

### groundzero-MCP-Server (11 Fehler)

**KRITISCH (4):**
1. health_tools.py: Missing `import os` und `from datetime import datetime`
2. system_tools.py: Missing `from datetime import datetime`
3. Ollama Container-Name falsch → read_logs() schlägt fehl
4. emergency_stops Table fehlt → SQL Error

**HOCH (3):**
5-6. String Escaping falsch: `"\\n"` → `"\n"` (health_tools.py, system_tools.py)
7. Async/Sync Mismatch: subprocess.run() in async def (blocking!)

**MITTEL (4):**
8-11. Hardcoded Pfade, fehlende Error-Handling, etc.

---

### docker-compose.yml (14 Fehler)

**KRITISCH (3):**
1. .env.example POSTGRES_URL falsch (localhost statt postgres)
2. .env.example REDIS_URL falsch (localhost statt redis)
3. prometheus-exporter Network falsch (n8n-network statt ground-zero-network)

**HOCH (5):**
4. Redis healthcheck: start_period fehlt
5. Ollama image: `latest` Tag (nicht reproduzierbar)
6. Ollama Security: cap_drop, security_opt, user fehlen
7. n8n healthcheck: /healthz Endpoint existiert möglicherweise nicht
8. docker-compose.mini.yml: Keine Security-Hardening

**MITTEL (6):**
9-14. Memory-Limits, Volume-Definitionen, Network-Optimierung, etc.

---

### n8n Workflows (12 Fehler)

**KRITISCH (5):**
1. WF-011 Line 53: `$ json` (Leerzeichen!) → ReferenceError
2. WF-011 Line 43: Handlebars `{{ }}` Syntax falsch
3. WF-003 Line 79: `$node` References bei sequenziellem Flow → Runtime CRASH
4. P1_WORKER Line 186: Invalid Error Workflow Reference
5. P1_CRITIC ↔ WORKER: Zirkuläre Abhängigkeit → INFINITE LOOP RISK

**HOCH (4):**
6-9. executeWorkflow IDs als Strings, Date Parsing Fehler, etc.

**MITTEL (3):**
10-12. Hardcoded Pfade, PostgreSQL Tables Existenz nicht verifiziert, etc.

---

### Shell Scripts (12 Fehler)

**KRITISCH (4):**
1. n-postgres-healthcheck.sh: SQL-Injection-Anfälligkeit (Zeilen 22, 26)
2. collect-logs.sh: Hardcodierter Git-Pfad funktioniert nur lokal
3. smoke-tests.sh: Unsichere Redis-Validierung (grep -q PONG zu permissiv)
4. smoke-tests.sh: docker volume inspect mit mehreren Args schlägt fehl

**HOCH (5):**
5-9. Kein Error-Handling, stat -c %Y nicht POSIX-konform, etc.

**MITTEL (3):**
10-12. Emoji-Encoding-Probleme, Container-Namen hardcoded, etc.

---

### Config Files (9 Fehler)

**KRITISCH (6):**
1. BACKUP_ENCRYPTION_KEY fehlt komplett (verwendet in pg-backup.sh)
2. Prometheus Exporter: 7 ENV-Variablen fehlen (DB_HOST, DB_PORT, etc.)
3. Deprecated Infisical Integration noch aktiv (pg-backup.sh, pg-restore.sh)
4. Claude Desktop MCP JSON: `"${GITHUB_TOKEN}"` wird nicht interpoliert
5. N8N_HOST Format falsch (localhost statt http://localhost:5678)
6. POSTGRES_HOST Ambiguität (localhost vs postgres)

**HOCH (1):**
7. SENTRY_DSN optional oder required? (inkonsistent)

**MITTEL (2):**
8-9. POSTGRES_INITDB_ARGS nicht dokumentiert, MCP Filesystem paths hardcoded

---

## 🎯 EURE AUFGABE ALS MULTI-AGENT-TEAM

### Rollen (intern abstimmen!)

**1. ARCHITECT / TECH LEAD**
- Dependency-Graph der 52 Fehler erstellen
- Critical Path identifizieren
- Deployment-Phasen definieren
- Risk-Assessment koordinieren

**2. BACKEND DEVELOPER (Python/FastAPI)**
- Agent Zero Service fixen (13 Fehler)
- MCP-Server fixen (11 Fehler)
- SQL-Queries korrigieren
- Unit-Tests schreiben

**3. DEVOPS ENGINEER**
- docker-compose.yml fixen (14 Fehler)
- Shell-Scripts sichern (12 Fehler)
- ENV-Variablen konsistent machen
- CI/CD Pipeline vorbereiten

**4. WORKFLOW ENGINEER (n8n Specialist)**
- n8n Workflows fixen (12 Fehler)
- JavaScript-Code debuggen
- Circular Dependencies auflösen
- Integration-Tests schreiben

**5. QA / TEST ENGINEER**
- Test-Strategy definieren
- Smoke-Tests erweitern
- E2E-Tests schreiben
- Rollback-Plan erstellen

**6. TECHNICAL WRITER / DOC ENGINEER**
- Deployment-Guide schreiben
- Troubleshooting-Dokumentation
- Runbook erstellen
- README aktualisieren

---

## 📦 DELIVERABLES (Was ihr liefern sollt)

### 1. MASTER IMPLEMENTATION PLAN (Haupt-Dokument)

**Format**: Markdown mit Mermaid-Diagrammen

**Struktur**:
```markdown
# 1. Executive Summary
- Projekt-Überblick (für Management)
- Status-Quo (was funktioniert, was nicht)
- Kritische Pfade (Blocker)
- Timeline-Übersicht (Gantt-Chart)
- Resource-Allocation (wer macht was)

# 2. Dependency Graph (Mermaid)
- Welche Fehler blockieren andere?
- Kritische Pfade visualisiert
- Parallele vs Sequenzielle Tasks

# 3. Task Breakdown
- Jeder Task: Max 2 Stunden
- Owner: Backend/DevOps/QA/etc.
- Dependencies: Was muss vorher fertig sein?
- Acceptance Criteria: Wann ist Task "Done"?

# 4. Timeline (Gantt-Chart)
- 2-Wochen-Sprint
- Milestones nach Tag 3, 7, 10
- Puffer-Zeiten einkalkuliert

# 5. Risk Assessment
- Jeder Task: Risk-Score 1-5
- Likelihood × Impact = Risk-Level
- Mitigation-Strategie für High-Risk

# 6. Testing Strategy
- Unit-Tests: Was testen?
- Integration-Tests: Welche Flows?
- E2E-Tests: Kompletter Datenfluss?
- Load-Tests: Optional (wenn Zeit)

# 7. Deployment Plan
- Phase 1: Blocker (Tag 1-3)
- Phase 2: High Priority (Tag 4-7)
- Phase 3: Medium + Docs (Tag 8-10)
- Rollout-Strategie: Lokal → Test → Production

# 8. Rollback Plan
- Was tun wenn Deployment schief geht?
- Wie rollen wir zurück?
- Daten-Recovery-Strategie
- Incident-Response-Plan

# 9. Monitoring & Alerting
- Welche Metriken tracken?
- Wann Alerts triggern?
- Healthcheck-Definition
- SLA-Definition (Uptime, Response Time)

# 10. Success Criteria
- Must-Have: Deployment-Ready
- Should-Have: Production-Quality
- Nice-to-Have: Optimierungen
```

---

### 2. FIX PATCHES (Konkrete Code-Fixes)

**Für jede der 5 Blocker-Issues:**

**Format**:
```markdown
## Blocker #1: SQL INDEX-Syntax

### Betroffene Datei:
`services/agent-zero/state_manager.py`

### Aktueller Code (Zeilen 71-89):
```python
[Fehlerhafte Code hier]
```

### Fix (Diff-Format):
```diff
- CREATE TABLE agent_zero_checkpoints (
-     id SERIAL PRIMARY KEY,
-     INDEX idx_task_id (task_id)
- );
+ CREATE TABLE agent_zero_checkpoints (
+     id SERIAL PRIMARY KEY
+ );
+ CREATE INDEX idx_task_id ON agent_zero_checkpoints (task_id);
```

### Testing:
- Unit-Test: `test_state_manager_init()`
- Verification: `docker exec ground-zero-postgres psql ... -c "\dt"`

### Risk-Level: LOW (isolierte Änderung)
```

**Patches für:**
1. SQL INDEX-Syntax (state_manager.py)
2. Missing Imports (health_tools.py, system_tools.py)
3. ENV-URLs (.env.example)
4. Ollama container_name (docker-compose.yml)
5. emergency_stops Table (init-postgres-schema.sql)

---

### 3. DEPLOYMENT RUNBOOK (Step-by-Step)

**Format**: Markdown mit Checklisten

**Struktur**:
```markdown
# Pre-Deployment Checklist
- [ ] Docker Desktop 4.50+ installed
- [ ] Git repository cloned
- [ ] All patches applied
- [ ] .env file created from .env.example
- [ ] All ENV variables set
- [ ] Ports 5432, 5678, 6379, 8000, 11434 available

# Phase 1: Blocker-Fixes (15 Minuten)
## Step 1: Apply SQL INDEX Fix
```bash
# Edit services/agent-zero/state_manager.py
# Apply patch #1
git diff services/agent-zero/state_manager.py
```

## Step 2: Add Missing Imports
[...]

## Step 3: Fix ENV-URLs
[...]

## Step 4-5: [...]

# Phase 2: Build & Start (10 Minuten)
```bash
# Build Agent Zero image
docker-compose build agent-zero

# Start all services
docker-compose up -d

# Wait for health
sleep 30
```

# Phase 3: Verification (10 Minuten)
```bash
# Run smoke tests
./scripts/smoke-tests.sh

# Check health
./scripts/n-postgres-healthcheck.sh

# Verify MCP
python groundzero-mcp-server/main.py --test
```

# Troubleshooting Guide
## Issue: PostgreSQL not starting
- Check: `docker logs ground-zero-postgres`
- Common cause: Port 5432 already in use
- Fix: [...]

## Issue: Agent Zero crash loop
- Check: [...]
- Fix: [...]

# Rollback Procedure
1. Stop all services: `docker-compose down`
2. Remove volumes: `docker volume rm postgres_data n8n_data`
3. Restore from backup: [...]
```

---

### 4. TEST PLAN (Quality Assurance)

**Unit-Tests** (Python):
```python
# tests/test_state_manager.py
async def test_create_checkpoint():
    """Test checkpoint creation in PostgreSQL"""

async def test_get_checkpoint():
    """Test checkpoint retrieval with caching"""

async def test_null_datetime():
    """Test NULL datetime handling (Bug #3)"""
```

**Integration-Tests** (n8n):
```markdown
## Test 1: Orchestrator → Worker Flow
- Trigger: POST to Orchestrator webhook
- Expected: Checkpoint created, Worker triggered
- Verify: PostgreSQL has checkpoint record

## Test 2: Worker → Agent Zero → Ollama
- Trigger: Complex reasoning task
- Expected: Agent Zero calls Ollama, result stored
- Verify: PostgreSQL has result with LLM response

## Test 3: Critic Validation with Retry
- Trigger: Worker with invalid output
- Expected: Critic fails, triggers retry (max 3x)
- Verify: PostgreSQL shows retry_count
```

**E2E-Test** (Claude → Result):
```markdown
## End-to-End Flow Test
1. Claude Desktop: Trigger MCP tool trigger_workflow()
2. n8n Orchestrator: Receives task
3. PostgreSQL: Checkpoint created
4. n8n Worker: Processes task
5. Agent Zero: (if complex) Calls Ollama
6. n8n Critic: Validates result
7. PostgreSQL: Final result stored
8. Claude Desktop: Receives notification

Duration: < 30 seconds
Success Criteria: Result visible in Claude Desktop
```

**Load-Tests** (Optional):
- 10 parallel tasks → no deadlocks
- 100 tasks in queue → all processed
- PostgreSQL connection pool → no exhaustion

---

### 5. RISK MATRIX & MITIGATION

| Risk | Likelihood | Impact | Score | Mitigation |
|------|-----------|--------|-------|------------|
| SQL INDEX fix breaks existing data | LOW (2) | HIGH (4) | 8 | Backup before migration, test on copy first |
| Docker networking issues | MEDIUM (3) | HIGH (4) | 12 | Test with docker-compose config validation |
| Circular dependency infinite loop | HIGH (4) | CRITICAL (5) | 20 | Add max_retries counter, timeout per workflow |
| Ollama memory exhaustion | MEDIUM (3) | MEDIUM (3) | 9 | Set memory limits, monitor with Prometheus |
| PostgreSQL connection pool exhaustion | LOW (2) | HIGH (4) | 8 | Increase pool size, add connection timeout |

**Mitigation Strategies**:
1. **Backup alles vor Changes** → Rollback möglich
2. **Graduelle Rollouts** → Erst lokal, dann test, dann prod
3. **Feature Flags** → Neue Features optional aktivierbar
4. **Circuit Breakers** → Ollama-Calls mit Timeout + Retry
5. **Monitoring** → Alerts bei kritischen Metriken

---

## 🔬 TECHNISCHE CONSTRAINTS & ARCHITEKTUR-REGELN

### Phase-Modell (WICHTIG!):
- **Phase 1 (JETZT)**: Nur lokale MCP-Server (Claude Desktop kann localhost erreichen)
- **Phase 2 (SPÄTER)**: Remote MCP via E2B (Claude Code Web braucht das)
- **Phase 3 (ZUKUNFT)**: OpenBao statt .env (Secrets-Management)

### Sicherheits-Anforderungen:
```yaml
# IMMER:
- Keine Ports exposed (nur 127.0.0.1)
- Non-root User in allen Containern (UID 3000+)
- cap_drop: ALL in docker-compose
- security_opt: no-new-privileges
- Secrets via .env (Phase 1), später OpenBao

# NIEMALS:
- 0.0.0.0 Port-Binding
- Root-User in Production
- Passwörter in Git
- SQL-Injection-Risiken
```

### Performance-Anforderungen:
- **PostgreSQL**: Source of Truth (nicht Redis!)
- **Redis**: Nur Cache (TTL 1h) + Queue
- **Distributed Locks**: Redis SETNX für Race Conditions
- **Connection Pooling**: asyncpg min=5, max=20
- **Response Time**: < 5s für einfache Tasks, < 30s für LLM

### Architektur-Regeln (aus CLAUDE.md):
1. **Stabilität vor Eleganz** → System muss funktionieren
2. **Checkpoints bei jedem Schritt** → Rollback-fähig
3. **Retries (3x)** → Error-Recovery automatisch
4. **Health-Checks** → Proaktives Monitoring
5. **Idempotente Operations** → Wiederholbar ohne Seiteneffekte

---

## ❓ KRITISCHE FRAGEN (Klärt diese im Plan!)

### Architektur-Entscheidungen:
1. **Task-Processing**: Parallel (höhere Throughput) oder Sequenziell (einfacher)?
2. **Checkpoint-Retention**: 7 Tage? 30 Tage? Automatische Cleanup?
3. **Dead-Letter-Queue**: Ja/Nein? Wenn ja, wie verarbeiten?
4. **Ollama GPU**: CPU (aktuell) oder GPU (Performance)? Kosten-Nutzen?
5. **State-Sync-Konflikte**: Wie detecten? Wie resolven? (PostgreSQL vs Redis)

### Deployment-Entscheidungen:
6. **Deployment-Strategie**: Lokaler Mini-Stack zuerst, oder direkt Full-Stack?
7. **Test-Kriterien**: Welche Tests MÜSSEN grün sein vor Deployment?
8. **Zero-Downtime**: Nötig? Wenn ja, wie? (Blue-Green, Rolling Update?)
9. **Feature-Flags**: Implementieren? Oder alle Features sofort live?
10. **Rollback-Trigger**: Wann rollt man zurück? (Fehlerrate? Latenz?)

### Monitoring-Entscheidungen:
11. **Metrics**: CPU, Memory, Disk → was noch? (Request Rate, Error Rate, Latency?)
12. **Alerts**: Wann triggern? (Memory > 80%? Error Rate > 5%?)
13. **Circuit-Breaker**: Für Ollama-Calls? (Timeout? Max Retries?)
14. **SLAs**: Uptime 99%? Response Time < 5s? Error Rate < 1%?
15. **n8n Critic**: Automatische Retries oder manuelle Intervention?

---

## 🗂️ REPOSITORY-INFORMATIONEN

### GitHub:
- **Repo**: https://github.com/24-firon/claude-agents
- **Branch**: `claude/remove-infisical-01VGnbAFqAmwHZn6hMkZeNRG`
- **Latest Commit**: `66bc846` (Review-Reports committed)

### Wichtigste Dateien (13 für Kontext):

**Projekt-Essenz:**
1. `CLAUDE.md` (Projekt-Gesetze, 11 Kapitel)
2. `README.md` (Quick-Start, Überblick)
3. `docs/ground-zero-v4.1/02-Master-v4.2.md` (Architektur)

**Implementierung:**
4. `docker-compose.yml` (5 Services)
5. `.env.example` (40+ ENV-Variablen)
6. `services/agent-zero/main.py` (FastAPI App)
7. `groundzero-mcp-server/main.py` (11 MCP-Tools)
8. `workflows/P1_ORCH_RouteTask_v1.json` (Orchestrator)

**Review-Reports:**
9. `CODE_REVIEW_GROUNDZERO_MCP_2025-11-19.md` (11 Fehler)
10. `DOCKER_COMPOSE_REVIEW_2025-11-19.md` (14 Fehler)
11. `N8N_WORKFLOWS_REVIEW_REPORT.md` (12 Fehler)

**Diagramme:**
12. `SKILLS-DEPENDENCY-DIAGRAM.md` (Skills-MCP-Matrix)
13. `docs/ground-zero-v4.1/00-INDEX-Dokumentations-Uebersicht.md` (Navigation)

---

## ⏱️ ZEITRAHMEN & RESOURCES

### Sprint-Details:
- **Dauer**: 2 Wochen (10 Arbeitstage)
- **Team**: 2 Personen (1 Backend-Dev + 1 DevOps)
- **Kapazität**: ~80 Stunden total (2 × 40h)

### Phasen-Aufteilung:
```
Phase 1 - Blocker-Fixes (Tag 1-3, 18-24h):
- Alle 5 Blocker behoben
- Tests geschrieben
- Lokal deploybar

Phase 2 - High-Priority (Tag 4-7, 30-40h):
- Alle 21 High-Issues behoben
- Integration-Tests
- Production-Ready

Phase 3 - Rest + Docs (Tag 8-10, 20-30h):
- Medium-Issues
- Dokumentation
- CI/CD Pipeline
- Final Testing
```

### Milestones:
- **Tag 3**: Lokaler Mini-Stack läuft fehlerfrei
- **Tag 7**: Full-Stack production-ready
- **Tag 10**: Dokumentation + CI/CD komplett

---

## 🎯 SUCCESS CRITERIA (Definition of Done)

### Must-Have (Phase 1 - Deployment-Ready):
- [ ] Alle 5 Blocker-Issues behoben & getestet
- [ ] `docker-compose up -d` startet ohne Fehler
- [ ] Alle 6 Smoke-Tests grün
- [ ] Health-Check zeigt "healthy" für alle 5 Services
- [ ] Claude Desktop kann groundzero-MCP-Tools aufrufen
- [ ] n8n Orchestrator-Workflow funktioniert (Webhook-Test)
- [ ] Agent Zero kann einfache Tasks verarbeiten
- [ ] PostgreSQL Checkpoints werden erstellt
- [ ] Logs sind lesbar (keine kritischen Errors)

### Should-Have (Phase 2 - Production-Quality):
- [ ] Alle 21 High-Priority-Issues behoben
- [ ] Integration-Tests: Orchestrator → Worker → Critic
- [ ] E2E-Test: Claude Desktop → n8n → Agent Zero → Result
- [ ] Backup-Verification läuft automatisch (täglich 03:00)
- [ ] Daily-Healthcheck läuft automatisch (täglich 02:00)
- [ ] Monitoring-Dashboard (Prometheus Metrics)
- [ ] Error-Recovery getestet (Retry-Mechanismus)
- [ ] Load-Test: 10 parallele Tasks funktionieren

### Nice-to-Have (Phase 3 - Optimization):
- [ ] Alle 52+ Issues behoben
- [ ] Performance optimiert (< 5s Response Time)
- [ ] Dokumentation vollständig (Runbook, Troubleshooting)
- [ ] CI/CD Pipeline läuft (GitHub Actions)
- [ ] Security-Audit bestanden
- [ ] Code-Coverage > 80%

---

## 🚀 START-PUNKT & VORGEHEN

### Schritt 1: Kontext Verstehen (1-2h)
1. Lest ALLE 13 kritischen Dateien
2. Versteht die Architektur (Datenfluss!)
3. Versteht die 52 Fehler (Dependencies!)

### Schritt 2: Dependency-Analysis (2-3h)
1. Erstellt Dependency-Graph (Mermaid)
2. Identifiziert Critical Path
3. Findet Quick-Wins

### Schritt 3: Plan Erstellen (3-4h)
1. Task-Breakdown (max 2h pro Task)
2. Timeline (Gantt-Chart)
3. Risk-Assessment
4. Testing-Strategy

### Schritt 4: Review & Refinement (1h)
1. Intern Review im Team
2. Lücken füllen
3. Risks absichern
4. Final Plan

**Dann: Plan an Auftraggeber schicken!**

---

## 💬 FRAGEN? UNKLARHEITEN?

Ihr habt Zugriff auf:
✅ Komplettes GitHub-Repository
✅ Alle 3 Review-Reports
✅ Komplette Architektur-Doku
✅ CLAUDE.md mit Projekt-Gesetzen
✅ Technischen Context dieser Briefing-Datei

**Bei Unklarheiten: Fragt!**

---

# 🎯 LOS GEHT'S - ERSTELLT DEN MASTER-PLAN!

**Erwartet: Ein detaillierter, abgestimmter Implementierungsplan als Markdown-Dokument mit allen oben genannten Deliverables.**

**Viel Erfolg!** 🚀
