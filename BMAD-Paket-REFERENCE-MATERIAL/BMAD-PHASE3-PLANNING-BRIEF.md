# BMAD PLANUNGSAUFTRAG: Ground-Zero Phase 3 - OpenBao Multi-Agent Secrets-Management

## 🎯 MISSION FÜR DEVELOPER-TEAM

Plant die **komplexeste Komponente** der Ground-Zero Infrastructure: Ein **Multi-Agent Secrets-Management System** mit OpenBao, das 22+ verschiedene Skills mit unterschiedlichen Permissions versorgt.

**Warum komplex?** Weil jeder Skill andere Secrets braucht, jeder Agent andere Rechte hat, und alles dynamisch + sicher + audit-compliant sein muss!

---

## 📚 KONTEXT: WAS IST GROUND-ZERO?

Ground-Zero ist eine **3-Pillar KI-Agentur-Infrastruktur** für autonome AI-Agents.

**Aktueller Stand (Phase 1-2 DONE):**
- ✅ Claude Desktop + MCP-Server (11 Tools)
- ✅ n8n Workflow-Orchestrator (5 Workflows)
- ✅ Agent Zero mit Ollama LLM
- ✅ PostgreSQL + Redis State-Management
- ✅ Docker-Compose (5 Services)

**Was FEHLT (Phase 3):**
- ❌ **OpenBao Secrets-Management** ← DAS PLANT IHR!
- ❌ Skill-Ecosystem Activation
- ❌ Multi-Tenant Support
- ❌ Compliance & Audit-Trails

---

## 🔐 DAS PROBLEM: SECRETS-CHAOS!

### Aktuell (Phase 1-2):
```bash
# .env File mit 40+ Secrets:
POSTGRES_PASSWORD=...
N8N_ENCRYPTION_KEY=...
ANTHROPIC_API_KEY=...
GITHUB_TOKEN=...
AWS_ACCESS_KEY_ID=...
# ... und 35 weitere!
```

**Probleme:**
1. **Alle Secrets in einer Datei** → ein Leak = alles weg!
2. **Keine Role-Based-Access** → jeder Skill sieht alle Secrets!
3. **Keine Rotation** → Passwörter nie geändert!
4. **Keine Audit-Trails** → wer hat wann was gelesen?
5. **Keine Dynamic Secrets** → DB-Credentials nie ablaufend!

---

## 🎯 DIE LÖSUNG: OpenBao Multi-Agent System

### Was ist OpenBao?
- **HashiCorp Vault Fork** (Open-Source)
- **Dynamic Secrets** - DB-Credentials mit TTL
- **Role-Based Access Control** - Jeder Agent nur seine Secrets
- **Audit-Logging** - Jeder Zugriff getracked
- **Encryption-as-a-Service** - Daten verschlüsseln ohne Key-Management

### Architektur-Vision:

```
┌─────────────────────────────────────────────────┐
│           22 SKILLS (verschiedene Agents)        │
├─────────────────────────────────────────────────┤
│ Droplet Diagnostics │ n8n Health │ Docker Cleanup│
│ Backup Validator    │ SLA Report │ Cloud Deploy │
│ ... 16 weitere Skills ...                        │
└──────────┬──────────────────────────┬────────────┘
           │                          │
           ▼                          ▼
    ┌─────────────┐           ┌─────────────┐
    │  MCP-Server │           │ Agent Zero  │
    │  (11 Tools) │           │  (FastAPI)  │
    └──────┬──────┘           └──────┬──────┘
           │                          │
           └───────────┬──────────────┘
                       ▼
              ┌─────────────────┐
              │    OPENBAO      │
              │ Secrets-Manager │
              ├─────────────────┤
              │ • Policies      │
              │ • Roles         │
              │ • Dynamic Creds │
              │ • Audit-Log     │
              └─────────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
┌──────────┐   ┌──────────┐   ┌──────────┐
│PostgreSQL│   │   AWS    │   │  GitHub  │
│ Dynamic  │   │ Dynamic  │   │   API    │
│  Creds   │   │  Creds   │   │  Tokens  │
└──────────┘   └──────────┘   └──────────┘
```

---

## 🧩 DAS SKILLS-ECOSYSTEM (22 Skills)

### Skill-Kategorien & Secret-Anforderungen:

**Infrastructure Skills (4 Skills):**
```yaml
Droplet Diagnostics v1.0:
  Secrets: SSH_KEY, DROPLET_IP, SSH_USER
  Permissions: READ-ONLY (Server-Metriken)

n8n Health Check v1.0:
  Secrets: N8N_API_KEY, N8N_URL
  Permissions: READ (Health), WRITE (Restart)

Repo Maintenance v1.0:
  Secrets: GITHUB_TOKEN (read:repo, write:repo)
  Permissions: FULL (Push commits)

Docker Cleanup v1.0:
  Secrets: DOCKER_HOST (optional)
  Permissions: WRITE (Container stop/remove)
```

**Data Skills (4 geplant):**
```yaml
Backup Status Validator:
  Secrets: AWS_ACCESS_KEY, AWS_SECRET, S3_BUCKET
  Permissions: READ-ONLY (S3-Backups listen)

SLA Reporting Engine:
  Secrets: POSTGRES_USER, POSTGRES_PASSWORD
  Permissions: READ-ONLY (Metrics-Queries)

Spec Validator:
  Secrets: None (File-System only)
  Permissions: READ (Specs), WRITE (Reports)

Cloud Deployer:
  Secrets: AWS_ACCESS_KEY (FULL), SSH_KEY
  Permissions: FULL (Deploy to Droplet)
```

**AI Skills (8 Anthropic Official):**
```yaml
skill-creator, mcp-builder, code-reviewer, github-pr:
  Secrets: ANTHROPIC_API_KEY
  Permissions: API-Calls (Rate-Limited per Skill)
```

**Custom Skills (6 User-Defined):**
```yaml
AI Agent Builder, CMO/Brand, etc.:
  Secrets: Varies (ANTHROPIC, OPENAI, Custom-APIs)
  Permissions: Custom Policies
```

---

## 🔥 EURE AUFGABE: PLANT DAS SYSTEM!

### Was ihr planen sollt:

**1. OpenBao Architektur-Design**
- Welche Secrets-Engines? (KV, Database, AWS, SSH)
- Welche Policies? (Per-Skill? Per-Agent? Per-Tenant?)
- Wie authentifizieren sich Skills? (AppRole? JWT? Kubernetes?)
- Wo läuft OpenBao? (Docker? Separate VM?)
- High-Availability? (3-Node-Cluster?)

**2. Policy-Design (Das Herzstück!)**
```hcl
# Beispiel-Policy für "Droplet Diagnostics" Skill:
path "secret/data/infrastructure/droplet/*" {
  capabilities = ["read"]
}
path "ssh/creds/droplet-readonly" {
  capabilities = ["read"]  # Dynamic SSH-Key
}
# ABER: Wie organisieren wir 22 Skills × N Secrets?
```

**3. Dynamic Secrets Integration**
- PostgreSQL: Credentials mit 24h TTL
- AWS: STS-Tokens mit 1h TTL
- SSH: Ephemeral Keys mit 15min TTL
- GitHub: Fine-grained Tokens mit Scope-Limits

**4. Skills → OpenBao Integration**
```python
# Jeder Skill braucht:
class Skill:
    def __init__(self):
        self.vault_client = get_vault_client()
        self.secrets = self.vault_client.read_secrets(
            path=f"skills/{self.name}"
        )

    def execute(self):
        # Benutze self.secrets['DROPLET_IP']
        pass
```

**5. Audit & Compliance**
- Wer hat wann welches Secret gelesen?
- Alerts bei ungewöhnlichen Zugriffen?
- Compliance-Reports (GDPR, SOC2)?
- Secret-Rotation-Policy?

**6. Migration von .env → OpenBao**
- Welche Secrets zuerst?
- Wie testen wir ohne Produktionsausfall?
- Rollback-Plan?
- Graduelle Migration über Feature-Flags?

---

## 📦 DELIVERABLES (Was ihr liefern sollt)

### 1. **OpenBao Architecture Design Document**
```markdown
# Struktur:
- System-Überblick (Diagramm)
- Secrets-Engines (welche für was?)
- Authentication-Methods (AppRole, JWT, etc.)
- Policy-Hierarchie (Skill → Role → Policy)
- Network-Topology (wo läuft OpenBao?)
- HA-Setup (Cluster-Config)
- Disaster-Recovery (Backup/Restore)
```

### 2. **Policy-Design Specification**
```markdown
# Für jeden Skill-Typ:
- Infrastructure Skills (4 Policies)
- Data Skills (4 Policies)
- AI Skills (8 Policies)
- Custom Skills (6 Policies)

# Pro Policy:
- Secrets-Paths (was darf gelesen werden?)
- Capabilities (read/write/delete/list?)
- Dynamic Secrets (DB, AWS, SSH?)
- TTL-Settings (wie lange gültig?)
- Renewal-Policy (auto-renew?)
```

### 3. **Skills Integration Plan**
```markdown
# Code-Änderungen pro Skill:
- Vault-Client-Initialization
- Secret-Fetching-Logic
- Error-Handling (Vault unavailable?)
- Caching-Strategy (Redis?)
- Testing-Strategy (Mock Vault?)
```

### 4. **Migration Roadmap (.env → OpenBao)**
```markdown
# 3-Phasen-Plan:
Phase 1 (Week 1): Critical Secrets (DB, API-Keys)
Phase 2 (Week 2): Non-Critical (SSH, AWS)
Phase 3 (Week 3): Dynamic Secrets (DB-Rotation)

# Pro Phase:
- Welche Secrets?
- Testing-Plan
- Rollback-Procedure
- Verification-Checklist
```

### 5. **Compliance & Audit Framework**
```markdown
# Audit-Requirements:
- Welche Events loggen? (read, write, rotate)
- Wo speichern? (PostgreSQL? S3?)
- Retention-Policy? (90 Tage? 1 Jahr?)
- Alerts? (3 failed logins → notify admin)
- Reports? (monatlich, wer hat was?)
```

### 6. **Implementation Plan (Timeline + Tasks)**
```markdown
# Gantt-Chart: 4-6 Wochen
Week 1: OpenBao Setup (Docker/VM)
Week 2: Policy-Design + Testing
Week 3: Skills-Integration (4 Core Skills)
Week 4: Migration Phase 1 (Critical)
Week 5: Migration Phase 2 + 3
Week 6: Testing + Docs + Rollout
```

---

## 🔍 TECHNISCHE CHALLENGES (Die komplexen Teile!)

### Challenge 1: Policy-Explosion Problem
**Problem:** 22 Skills × 5 Secrets = 110 Policies?
**Frage:** Wie organisieren? (Gruppieren? Inheritance? Templates?)

### Challenge 2: Dynamic Secrets Lifecycle
**Problem:** PostgreSQL-Credentials mit 24h TTL - wie renew?
**Frage:** Skills müssen refreshen - wann? automatisch? Error-Handling?

### Challenge 3: Vault Unavailability
**Problem:** OpenBao down → alle Skills kaputt!
**Frage:** Fallback? Cache? Graceful Degradation?

### Challenge 4: Secret-Versioning
**Problem:** API-Key rotiert → alte Version noch 24h gültig?
**Frage:** Wie handlen wir Versionen? (Blue-Green für Secrets?)

### Challenge 5: Multi-Tenant Isolation
**Problem:** 2 Clients → beide nutzen "Droplet Diagnostics" Skill
**Frage:** Wie isolieren? (Separate Vaults? Namespaces? Policies?)

---

## 📊 REFERENZEN (Was ihr lesen sollt)

### Dateien im Repo die ihr BRAUCHT:

**Must-Read (7 Dateien):**
1. `CLAUDE.md` - Projekt-Gesetze
2. `docs/ground-zero-v4.1/02-Master-v4.2.md` - Architektur
3. `openbao-agents-complete-guide.md` - OpenBao-Guide (785 Zeilen!)
4. `SKILLS-ECOSYSTEM-COMPLETE-ANALYSIS.md` - Alle 22 Skills
5. `SKILLS-DEPENDENCY-DIAGRAM.md` - Skills-MCP-Matrix
6. `.env.example` - Aktuelle 40+ Secrets
7. `docker-compose.yml` - Wo OpenBao reinpasst

**Nice-to-Have (3 Dateien):**
8. `docs/ground-zero-v4.1/03-IMPLEMENTATION-ROADMAP.md` - Phase 3 Overview
9. `docs/operations/DISASTER-RECOVERY-PLAYBOOK.md` - Backup-Strategy
10. `skills/HANDBUCH.md` - Skills-Manual

---

## 🎯 KRITISCHE FRAGEN DIE IHR BEANTWORTEN MÜSST:

### Architektur:
1. **Wo läuft OpenBao?** Docker-Container? Separate VM? Cloud?
2. **High-Availability?** Single-Node? 3-Node-Cluster? Auto-Failover?
3. **Storage-Backend?** Filesystem? PostgreSQL? Consul?
4. **Network-Security?** Intern only? TLS? mTLS? Firewall-Rules?

### Policies:
5. **Gruppen-Policies?** Skills nach Kategorie? (Infrastructure, Data, AI)
6. **Inheritance?** Base-Policy + Skill-Specific?
7. **Dynamic-Policy-Generation?** Per Skill auto-generiert?
8. **Policy-Versioning?** Wie updaten ohne Downtime?

### Integration:
9. **Skill-Authentication?** AppRole? JWT? Kubernetes-Auth?
10. **Secret-Caching?** Redis? In-Memory? TTL?
11. **Error-Handling?** Fallback zu .env? Cache? Fail-Fast?
12. **Testing?** Mock-Vault für Unit-Tests? Test-Vault?

### Operations:
13. **Unsealing?** Auto-Unseal (Cloud-KMS)? Manual? Shamir?
14. **Backup?** Snapshot-Schedule? S3? Encryption?
15. **Monitoring?** Metrics? Alerts? Dashboards?
16. **Secret-Rotation?** Automatisch? Manual? Schedule?

---

## ⏱️ ZEITRAHMEN & COMPLEXITY

**Projekt-Größe:** GROSS (6-8 Wochen mit 2 Leuten)

**Complexity-Level:** HOCH
- Viele Moving Parts (22 Skills, OpenBao, Policies)
- Security-Critical (ein Fehler = alle Secrets exposed)
- Migration ohne Downtime (Production läuft!)
- Testing-Challenge (wie testet man Secrets-Management?)

**Team-Empfehlung:**
- 1 Security-Engineer (OpenBao-Expertise)
- 1 Backend-Developer (Skills-Integration)
- 1 DevOps (Infrastructure + Migration)
- 1 QA (Security-Testing + Compliance)

---

## 🎯 SUCCESS CRITERIA

### Must-Have:
- [ ] OpenBao läuft (Docker oder VM)
- [ ] Alle 22 Skills haben Policies
- [ ] 4 Core Skills integriert (Droplet, n8n, Repo, Docker)
- [ ] Dynamic Secrets für PostgreSQL funktionieren
- [ ] Migration von .env → OpenBao für kritische Secrets
- [ ] Audit-Log funktioniert
- [ ] Backup/Restore getestet

### Should-Have:
- [ ] Alle 22 Skills integriert
- [ ] HA-Setup (3 Nodes)
- [ ] Auto-Unseal mit Cloud-KMS
- [ ] Compliance-Reports automatisiert
- [ ] Monitoring-Dashboard

### Nice-to-Have:
- [ ] Multi-Tenant Support
- [ ] Secret-Versioning
- [ ] Blue-Green Secret-Rotation
- [ ] Automated Compliance-Checks

---

## 🚀 START-PUNKT

**Beginnt mit:**
1. Lest `openbao-agents-complete-guide.md` (785 Zeilen Goldmine!)
2. Lest `SKILLS-ECOSYSTEM-COMPLETE-ANALYSIS.md` (alle 22 Skills verstehen)
3. Analysiert `.env.example` (welche Secrets sind wo?)
4. Entwerft High-Level Architektur (Diagramm)
5. Erstellt Policy-Matrix (22 Skills × Secrets)

**Dann: Detailplan erstellen!**

---

# 💬 FRAGEN?

Ihr habt Zugriff auf:
- ✅ GitHub-Repository
- ✅ Alle Architektur-Docs
- ✅ OpenBao-Guide (785 Zeilen!)
- ✅ Skills-Ecosystem-Analyse

**BEI UNKLARHEITEN: FRAGT!**

---

# 🎯 LOS GEHT'S - PLANT DAS SECRETS-MANAGEMENT-SYSTEM!

**Das ist DAS komplexe Teil von Ground-Zero Phase 3!** 🔐🚀
