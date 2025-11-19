# 06 – Skills & Agents: Dein Werkzeugkasten

**Version:** 1.0  
**Phase:** 1-2 (Skills in Phase 1, Agents produktiv in Phase 2)  
**Säule:** A (Desktop), aber wirkt auf B  
**Datum:** 2025-11-19  

---

## 🧠 Was „Skills" in Ground-Zero genau sind

In deinen Texten sind Skills keine vagen „Prompts", sondern **versionierte Markdown-Artefakte**, die das Verhalten von Claude Desktop/Claude Code präzise definieren.

Ein Skill hat laut `erkenntnisse-komplett` und Tool-Landscape immer eine **YAML-Frontmatter** (z.B. `name`, `description`) und darunter klar formulierte Anweisungen, oft mit progressiver Offenlegung, damit nur bei Bedarf der volle Text in den Kontext geladen wird.

Das Design-Prinzip dahinter ist **Token-Effizienz**: Skills sind so gebaut, dass das Modell selbst entscheidet, wann ein Skill „gezogen" wird, und so je nach Aufgabe bis zu etwa 73 % Prompt-Overhead einspart im Vergleich zu ständigem manuellen Copy-Paste.

Speicherorte sind explizit: lokal unter `.claude/skills` (persönliche Skills) und projektbezogen in `.claude/skills` im Repo, damit Skills wie Code versionierbar sind und in Branches/Phasen sauber mitwandern.

### 🔁 Veranschaulichung – Anatomie eines Skills

```text
SKILL-Datei (Markdown)
┌─────────────────────────────────────────────┐
│ YAML-Frontmatter                           │
│ name: "Droplet Diagnostics Skill"          │
│ version: "1.0"                             │
│ description: "Führt D1-Diagnose-Skripte..."│
│ ...                                        │
├─────────────────────────────────────────────┤
│ Markdown-Körper                            │
│ - Erklärung, wann Skill zuständig ist      │
│ - Beispiel-Prompts                         │
│ - Hinweise auf erlaubte Befehle/Pfade      │
└─────────────────────────────────────────────┘
```

---

## 🧩 Skill-Ökosystem: welche Skill-Typen du vorgesehen hast

In `Tool-Landschaft Essentials` und den Skill-Blueprints taucht ein kleines „Ökosystem" von Skill-Typen auf, das später eng mit MCP, E2B und den Droplet-Skripten zusammenspielt.

Du unterscheidest im Projekt mehrfach zwischen **„Desktop-Skills"** (für lokale/Operator-Aufgaben), **MCP-bezogenen Skills** (z.B. `mcp-builder`) und **Ops-Skills**, die direkt auf Scripts wie `droplet-full-diagnostics.sh` oder `automation-helpers.sh` aufsetzen.

Im Projektdaten-Kontext werden außerdem konkrete System-Skills wie `skill-creator`, `mcp-builder`, `ai-agent-builder` und ein CMO-Skill erwähnt, mit der offenen Frage, ob diese schon aktiv sind oder in Phase 1 noch eingerichtet werden müssen.

Die Default-Annahme in deinen Notizen lautet ausdrücklich: **Diese Skills sind in Phase 1 noch einzurichten**, gestützt durch einen Verweis auf `phase1/desktop-and-e2b/claude-code-skills-setup.md`.

### 📊 Veranschaulichung – Skill-Typen

| Skill-Typ | Beispiel | Aufgabe | Phase |
|----------|----------|---------|-------|
| Desktop-Ops-Skill | Droplet Diagnostics | D1-Diagnose, Health-Analyse | 2 (auf Basis Vorarbeit in 1) |
| Desktop-Ops-Skill | n8n Health | n8n/Postgres-Gesundheit prüfen | 2 |
| Repo-Ops-Skill | Repo Maintenance | Git-Status, Hygiene, keine Autocommits | 2 |
| Docker-Ops-Skill | Docker Cleanup | Disk-Nutzung analysieren, Cleanup-Plan vorschlagen | 2 |
| System-Skill | `skill-creator` | neue Skills erzeugen | 1 (Einrichtung) |
| System-Skill | `mcp-builder` | MCP-Server manifestieren/aufsetzen | 1–2 |
| System-Skill | `ai-agent-builder` | Agent-Workflows aus Skills/Skripten bauen | 2–3 |

---

## 🔧 Vier Kern-Blueprints: deine Ops-Skills im Detail

In `skill-blueprints.md` definierst du vier ganz konkrete Ops-Skills als Blueprints, jeweils mit Zweck, erlaubten Pfaden, erlaubten Kommandos, Inputs, Outputs und Guardrails.

Diese vier Skills sind als „Ops-Werkzeugkasten" für Ground-Zero gedacht: Diagnostik, Health-Check, Repo-Pflege und Docker-Aufräumplanung.

### 1️⃣ Droplet Diagnostics Skill

Der Droplet-Diagnostics-Skill soll automatisiert deine D1-Diagnose-Skripte ausführen, Output sammeln und anhand einer Checkliste eine priorisierte To-do-Liste erzeugen.

Use-Cases: regelmäßiger Gesundheitscheck des Droplets, tiefere Analyse bei Störungen und längerfristige Trend-Beobachtung für CPU/RAM/Disk.

**Wichtige Punkte aus dem Blueprint:**
- Read-Access nur auf `scripts/analysis/droplet-full-diagnostics.sh`, `n8n-postgres-healthcheck.sh`, Diagnose-Checkliste und frühere Outputs.
- Write-Access nur in eigene Diagnose-Output- und Report-Ordner, keine Produktivdaten.
- No-Access explizit für `.env`, Secrets, produktive DB-Daten.

### 📋 Veranschaulichung – Droplet-Diagnostics-Scope

```text
Erlaubte Pfade (READ):
  - scripts/analysis/droplet-full-diagnostics.sh
  - scripts/analysis/n8n-postgres-healthcheck.sh
  - spec/infra/droplet-diagnostics-checklist.md
  - diagnostics-output/*

Erlaubte Pfade (WRITE):
  - diagnostics-output/*
  - diagnostics-output/reports/*

Verboten:
  - .env, .env.mcp
  - /var/lib/postgresql/data
  - produktive n8n-Workflows/Secrets
```

### 2️⃣ n8n Health Skill

Der n8n-Health-Skill zielt auf **Gesundheitschecks für n8n + Postgres**: Container-Status, Ports, Health-Endpoints (`/healthz`), typische Fehlerbilder in Logs.

Er nutzt u.a. `n8n-postgres-healthcheck.sh`, Infrastruktur-Specs (z.B. `spec/infra/n8n-postgres-stack.yaml`) und die Helper-Funktionen aus `automation-helpers.sh`.

**Charakteristika im Blueprint:**
- Fokussiert auf Liveness/Readiness, nicht auf inhaltliche Workflow-Logik.
- Dient als „First Responder": sind Container oben, kann DB reden, ist Port offen, tauchen Fehler im n8n-Log auf.

### 3️⃣ Repo Maintenance Skill

Der Repo-Maintenance-Skill kümmert sich um **Git-Status, Pull-Hygiene und einfache Repo-Checks**, ohne jemals destructive Git-Kommandos auszuführen.

Er baut auf `automation-helpers.sh`-Funktionen wie `syncrepo` und `repostatus` auf, um den Zustand des `claude-agents`-Repos lesend zu inspizieren.

**Zentrale Aspekte:**
- Anzeigen von `git status`, letzten Commits und Remote-Branches.
- Warnen bei uncommitted Changes („erst committen/stashen, dann synchronisieren").
- Kein automatisches `git add/commit/push` – alles bleibt Vorschlagsebene.

### 4️⃣ Docker Cleanup Skill

Der Docker-Cleanup-Skill ist bewusst als **„Analyse- und Vorschlags-Skill, keine Auto-Execution"** designt.

Er ruft z.B. `docker system df -v` auf, zeigt Container/Images/Volumes und schlägt konkrete `docker container/image/volume/system prune`-Kommandos vor – führt sie aber nicht selbst aus.

Im Blueprint sind klare Warnhinweise hinterlegt: Der Skill soll **nur einen Plan** für dich oder einen höherstufigen Agenten liefern, nicht eigenmächtig Ressourcen löschen.

---

## 🛠️ automation-helpers.sh – der Skill-Unterbau

`automation-helpers.sh` ist eine zentrale Script-Datei, die du explizit **als Helper-Schicht für Skills und manuelle Automatisierung** definierst.

Sie kapselt SSH-Verbindungen, Repo-Syncs, Droplet-Diagnosen, n8n-Health-Checks, lokale Docker-Statusabfragen und nicht-destruktive Backup/Cleanup-Vorschläge unter klar benannten Funktionen.

Wichtige Eigenschaften:
- **Nicht destruktiv**: keine automatischen Commits, keine Container-Reboots, keine Docker-Prunes – nur lesende Observierung oder Vorschläge.
- Klar farbcodierte Log-Ausgaben (`loginfo`, `logsuccess`, `logwarning`, `logerror`) für Mensch + Skill-Parsing.
- Zentrale Konfiguration am Anfang (`DROPLETHOST`, `DROPLETUSER`, `LOCALREPOPATH`, etc.), damit du Umgebungen wechseln kannst, ohne Skills umzuschreiben.

### 📊 Veranschaulichung – Funktionsübersicht automation-helpers

| Kategorie | Beispiel-Funktionen | Zweck |
|----------|---------------------|-------|
| Repo | `syncrepo`, `repostatus` | Git-Status & Pull ohne destruktive Aktionen |
| Droplet | `rundiagnostics`, `runn8nhealth`, `fetchremotefile` | D1-Diagnose & n8n-Health über SSH |
| Docker lokal | `dockerstatus`, `dockerlogs`, `dockerstatssnapshot` | Überblick Container/Images/Volumes/Stats |
| Backup | `backupdiagnostics` | Tar-Backup der Diagnostik-Outputs |
| Cleanup-Vorschlag | `suggestdockercleanup`, `suggestolddiagnosticscleanup` | Nur Vorschläge, kein Auto-Delete |
| System | `systemresources`, `showhelp` | CPU/RAM/Disk-Snapshot & Funktionsübersicht |

---

## 🛡️ Sicherheits- und Missbrauchs-Schutz für Skills

In Blueprints und Helper-Script ziehst du sehr harte Linien, um Skill-Missbrauch zu verhindern.

Typische Guardrails:
- **Whitelists für Pfade**: Skills dürfen nur in klar definierten Verzeichnissen lesen/schreiben (`scripts/analysis`, `diagnostics-output`, `backups`, keine `.env`, keine DB-Volumes).
- **Whitelists für Befehle**: Erlaubte Kommandos sind genau aufgelistet (z.B. `droplet-full-diagnostics.sh`, `n8n-postgres-healthcheck.sh`, `docker system df`, `git status`), alles andere ist Tabu.
- **Keine Autoprunes / keine Shell-Execs ohne dich**: Cleanup-Befehle werden nur als Text vorgeschlagen, nie direkt ausgeführt; Git-Befehle bleiben auf read-only oder safe Pull beschränkt.
- **Exit-Codes + Logs** in Backup/DR-Skripten, damit Skills später deterministisch erkennen können, ob etwas geklappt hat oder nicht.

---

## 🧑‍💻 Agents-Schicht: wie Agents mit Skills zusammenspielen

In den Launch-Vorbereitungen und Meta-Plänen tauchen mehrere Agent-Rollen und das File `AGENTS.md` auf, ergänzt um einen `ground-zero-agent-kontext-phase1`.

Parallel dazu beschreiben deine Multi-Instanz-Prompts klare Spezialisierungen: GitHub-Runner-Agent, Skills-Engineer, Backup/DR-Architektin, n8n-Compliance-Runbook-Architekt – alles „Agentenrollen", die auf Skills + Scripts aufsetzen.

Typische Agent-Pattern:
- **GitHub-Runner-Agent**: entwirft Workflows (`.github/workflows/*.yml`) und Dokus (`CLAUDE-RUNNER-SETUP.md`) für CI-Jobs, fasst sich aber nicht an Git-Push oder Livesysteme.
- **Skills-Engineer**: baut aus `skill-blueprints.md` konkrete Skill-Verzeichnisse mit `SKILL.md`, Manifest, README und INSTALL/TEST-Playbook.
- **Backup/DR-Agentin**: schreibt `pg-backup.sh`, `pg-restore.sh`, `backup-validation.sh` sowie ein `DISASTER-RECOVERY-RUNBOOK.md` – alles als Vorschlag, nicht als live ausgeführte Aktionen.
- **n8n-Compliance-Agent**: entwirft `n8n-WORKFLOW-LIBRARY.md` und GDPR/SLA-Runbooks, die später teilweise als Skills oder n8n-Workflows umgesetzt werden.

### 🔁 Veranschaulichung – Agent → Skill → Script-Kette

```text
[Agent: Backup/DR]
      |
      | 1) liefert: pg-backup.sh, DR-Runbook.md (Textvorschlag)
      v
[Du + Git]
      |
      | 2) reviewst, passt an, commit/push
      v
[Skill: Backup-Validation oder Ops-Skill]
      |
      | 3) ruft Skript kontrolliert über MCP/E2B/Desktop auf
      v
[System: Droplet + Postgres + n8n]
      |
      | 4) schreibt Logs/Exit-Codes
      v
[Monitoring/Compliance-Ebene wertet aus]
```

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige Skills-/Agents-Beschreibung |

---

*Lies als nächstes `07-n8n-Workflows-und-Automation.md` für Details zur Workflow-Library.*
