# BMAD PLANUNGSAUFTRAG: Ground-Zero Phase 3/4 - Agent Memory & Knowledge-Base System

## 🧠 MISSION FÜR DEVELOPER-TEAM

Plant ein **Multi-Layer Agent Memory System** mit:
- **Short-Term Memory** (aktive Tasks, Context-Window)
- **Long-Term Memory** (Skills-Erfahrungen, Learnings)
- **Knowledge-Base** (Doku, Code-Patterns, Best-Practices)
- **Memo-System** (Agents schreiben Notizen für andere Agents)
- **RAG-Integration** (Retrieval-Augmented Generation)

**Warum komplex?** Weil 22 Agents gleichzeitig Memory brauchen, Wissen teilen müssen, Context-Windows managen, und alles performant + searchable + versioniert sein muss!

---

## 📚 KONTEXT: WARUM BRAUCHEN WIR DAS?

### Das Problem (aktuell):
```
Agent startet Task
  → Hat KEIN Memory von vorherigen Tasks!
  → Macht denselben Fehler wieder!
  → Kann nicht von anderen Agents lernen!
  → Context-Window voll nach 3 Tasks!
```

### Die Vision:
```
Agent startet Task
  → Lädt relevante Memories aus Memory-DB
  → Sieht Memos von anderen Agents
  → Nutzt Knowledge-Base für Best-Practices
  → Speichert Learnings nach Task
  → Andere Agents profitieren davon!
```

---

## 🏗️ DAS SYSTEM: 5 MEMORY-EBENEN

### **Ebene 1: Short-Term Memory (Redis)**
```
Aktive Task-Informationen:
- Aktueller Task-Context
- Letzte 10 Messages
- Temporäre State-Variables
- TTL: 1-24 Stunden
```

**Beispiel:**
```json
{
  "agent_id": "agent-zero-worker-01",
  "task_id": "task-12345",
  "context": {
    "conversation": ["User: Fix bug", "Agent: Analyzing..."],
    "current_file": "main.py",
    "last_action": "read_file",
    "variables": {"bug_type": "SQL-Injection"}
  },
  "ttl": 3600
}
```

---

### **Ebene 2: Long-Term Memory (PostgreSQL)**
```
Persistente Agent-Erfahrungen:
- Tasks-History (alle Tasks ever)
- Skill-Usage-Patterns
- Error-Learnings
- Success-Patterns
- Retention: Unbegrenzt (oder 1 Jahr?)
```

**Schema:**
```sql
CREATE TABLE agent_memories (
  id SERIAL PRIMARY KEY,
  agent_id VARCHAR(255),
  skill_name VARCHAR(255),
  task_type VARCHAR(100),
  context_snapshot JSONB,
  result_type VARCHAR(50),  -- success/failed/timeout
  learnings TEXT,            -- Was gelernt?
  created_at TIMESTAMP
);

CREATE INDEX idx_agent_skill ON agent_memories (agent_id, skill_name);
CREATE INDEX idx_task_type ON agent_memories (task_type);
```

---

### **Ebene 3: Knowledge-Base (Vector DB - pgvector)**
```
Searchable Dokumentation & Code:
- Ground-Zero Doku (alle MD-Files)
- Code-Patterns (bewährte Lösungen)
- API-Docs (n8n, PostgreSQL, etc.)
- Embeddings für Semantic-Search
```

**Beispiel:**
```python
# Query:
query = "How to create PostgreSQL checkpoint?"

# Semantic-Search:
results = vector_db.search(query, limit=5)
# Returns:
# - docs/ground-zero-v4.1/02-Master-v4.2.md (Sektion 3.2)
# - services/agent-zero/state_manager.py (Funktion: create_checkpoint)
# - ERKENNTNISSE_KOMPLETT.md (Learning #42)
```

---

### **Ebene 4: Memo-System (PostgreSQL + Notifications)**
```
Agent-to-Agent Kommunikation:
- Agent A schreibt Memo für Agent B
- Prioritäten (urgent/normal/low)
- Expiry-Dates
- Read-Receipts
```

**Schema:**
```sql
CREATE TABLE agent_memos (
  id SERIAL PRIMARY KEY,
  from_agent_id VARCHAR(255),
  to_agent_id VARCHAR(255),     -- NULL = alle Agents
  subject VARCHAR(500),
  content TEXT,
  priority VARCHAR(20),
  tags VARCHAR[],               -- ["bug", "deployment"]
  expires_at TIMESTAMP,
  read_at TIMESTAMP,
  created_at TIMESTAMP
);
```

**Beispiel-Memo:**
```json
{
  "from": "agent-zero-critic",
  "to": "agent-zero-worker-*",  // alle Worker
  "subject": "SQL-Injection-Pattern erkannt!",
  "content": "In Task task-12345 wurde SQL-Injection gefunden. Pattern: user_input direkt in Query. FIX: Parameterized Queries verwenden!",
  "priority": "urgent",
  "tags": ["security", "sql", "best-practice"],
  "expires_at": null  // nie ablaufend
}
```

---

### **Ebene 5: Skill-Knowledge (JSON Files + Git)**
```
Per-Skill Konfiguration & Learnings:
- Skill-Configs (welche MCP-Server?)
- Usage-Statistics
- Known-Issues
- Best-Practices
- Versioniert in Git
```

**Struktur:**
```
skills/
├── droplet-diagnostics/
│   ├── config.json
│   ├── knowledge.md          # Learnings
│   ├── patterns.yaml         # Bewährte Patterns
│   └── issues.md             # Known Issues
├── n8n-health-check/
│   └── ...
└── ... (22 Skills)
```

---

## 🔄 DATENFLUSS: WIE MEMORY FUNKTIONIERT

### **Scenario 1: Agent startet neuen Task**
```
1. Agent empfängt Task (z.B. "Deploy to Droplet")
2. Agent fragt Memory-System:
   a. Short-Term: "Gibt es aktiven Task für diesen Droplet?"
   b. Long-Term: "Welche Deploy-Tasks waren erfolgreich?"
   c. Knowledge-Base: "Was steht in der Deploy-Doku?"
   d. Memos: "Gibt es Warnungen von anderen Agents?"
   e. Skill-Knowledge: "Cloud-Deployer Best-Practices?"

3. Agent erhält Context-Bundle:
   {
     "short_term": null,
     "long_term": [
       {"task_id": "task-999", "result": "success", "pattern": "use SSH-key-auth"}
     ],
     "knowledge": [
       {"doc": "02-Master-v4.2.md", "section": "Deployment", "relevance": 0.95}
     ],
     "memos": [
       {"from": "agent-critic", "subject": "SSH-Permissions-Issue!", "priority": "urgent"}
     ],
     "skill_patterns": {
       "recommended_approach": "Ansible-Playbook",
       "common_errors": ["Permission-denied", "Firewall-blocked"]
     }
   }

4. Agent verarbeitet Task MIT Context

5. Nach Task-Abschluss:
   a. Short-Term: Cache Task-Result (1h)
   b. Long-Term: Speichere Learning:
      "Deploy mit SSH-key-auth funktioniert, aber Firewall muss geöffnet sein"
   c. Memo schreiben (optional):
      "An alle: Droplet-Firewall für Port 22 öffnen vor Deploy!"
   d. Skill-Knowledge updaten:
      "cloud-deployer: +1 successful deploy, add to patterns.yaml"
```

---

### **Scenario 2: Agent schlägt fehl (Error-Learning)**
```
1. Agent: Deploy-Task schlägt fehl (Timeout)

2. Agent speichert in Long-Term:
   {
     "result": "failed",
     "error": "SSH-Connection-Timeout",
     "learnings": "Firewall blockiert Port 22",
     "recommended_fix": "doctl compute firewall add-rule ..."
   }

3. Agent schreibt Memo:
   "Von: agent-zero-deploy
    An: alle
    Betreff: Deploy-Fehler auf Droplet XYZ
    Inhalt: SSH-Timeout → Firewall checken!
    Tags: [deployment, firewall, ssh]"

4. Nächster Agent (30min später):
   - Lädt Long-Term Memory
   - Sieht failed Task
   - Liest Memo
   - Öffnet Firewall ZUERST
   - Deploy erfolgreich!

5. Learning für alle zukünftigen Deploys gespeichert!
```

---

## 🎯 EURE AUFGABE: PLANT DAS MEMORY-SYSTEM!

### Was ihr planen sollt:

**1. System-Architektur**
- Welche Datenbank für was? (Redis/PostgreSQL/pgvector)
- Wie integriert sich das mit bestehendem System?
- Performance? (1000 Agents gleichzeitig?)
- Storage-Growth? (1 Jahr = wieviel GB?)

**2. Memory-Retrieval-Strategie**
```python
# Wie findet Agent relevante Memories?
class MemoryRetriever:
    def get_relevant_context(self, task):
        # 1. Semantic-Search in Knowledge-Base?
        # 2. Filter Long-Term nach task_type?
        # 3. Ranking-Algorithm? (Relevance-Score?)
        # 4. Max Context-Size? (nicht mehr als 50K Tokens?)
        pass
```

**3. RAG-Integration (Retrieval-Augmented Generation)**
```
Problem: Agent braucht Doku-Kontext, aber 200K Tokens sind zu viel!
Lösung: Embedding-basierte Semantic-Search

Workflow:
1. User-Query → Embedding (via OpenAI/local)
2. Semantic-Search in pgvector (Top-5 Docs)
3. Nur relevante Docs an Agent senden
4. Agent generiert Antwort mit Context
```

**4. Memo-Notification-System**
```
Problem: Agent B muss Memo von Agent A sehen - aber wann?
Optionen:
  A) Polling (Agent fragt alle 10s nach Memos)
  B) Push (PostgreSQL NOTIFY/LISTEN)
  C) Webhook (Agent registriert Webhook-URL)

Welche Option? Trade-offs?
```

**5. Context-Window-Management**
```
Problem: Agent hat 200K Token-Limit, aber:
  - 50K Tokens aktuelle Conversation
  - 30K Tokens Long-Term Memories
  - 40K Tokens Knowledge-Base Docs
  - 20K Tokens Memos
  = 140K Tokens → passt!

Aber: Wie priorisieren wenn zu viel?
  - Relevance-Ranking?
  - Recency-Weighting?
  - Task-Type-Filter?
```

**6. Memory-Cleanup & Retention**
```
Fragen:
- Short-Term: TTL 1h? 24h?
- Long-Term: Für immer? 1 Jahr? Automatische Archivierung?
- Knowledge-Base: Versions-Control? (Git?)
- Memos: Expire nach 30 Tagen? Read-once?
- Skill-Knowledge: Git-Commits für jedes Learning?
```

**7. Multi-Tenant Isolation**
```
Problem: 2 Clients → beide nutzen "Droplet Diagnostics"
→ Dürfen sie sich gegenseitig Memories sehen?

Lösung-Optionen:
  A) Separate Memory-DBs per Tenant
  B) Row-Level-Security in PostgreSQL
  C) Namespace-Prefixing (tenant_id in allen Keys)
```

**8. Performance-Optimierung**
```
Challenges:
- 1000 Agents fragen gleichzeitig Memory-System
- Semantic-Search muss < 100ms sein
- Long-Term-Queries dürfen DB nicht blockieren

Lösungen:
- Read-Replicas für PostgreSQL?
- Caching-Layer (Redis)?
- Async-Queries (nicht blockierend)?
- Connection-Pooling?
```

---

## 📦 DELIVERABLES

### 1. **Memory-System Architecture Document**
```markdown
# Struktur:
- System-Überblick (5-Ebenen-Diagramm)
- Data-Flow (Task-Start bis Memory-Save)
- Database-Schemas (alle Tables)
- RAG-Integration (Embedding-Pipeline)
- Notification-System (Memos)
- Performance-Considerations
- Security & Privacy
```

### 2. **Database-Schema-Design**
```sql
-- Alle Tables für Memory-System:
-- agent_memories (Long-Term)
-- agent_memos (Agent-to-Agent)
-- knowledge_embeddings (pgvector)
-- skill_knowledge (per-Skill)
-- memory_access_log (Audit)
```

### 3. **RAG-Pipeline-Specification**
```markdown
# Komponenten:
- Embedding-Model (welches? OpenAI? local?)
- Vector-Store (pgvector vs Pinecone vs Qdrant?)
- Chunking-Strategy (wie große Docs splitten?)
- Reranking (nach Semantic-Search?)
- Caching (häufige Queries?)
```

### 4. **Memory-API-Design**
```python
# Python-API für Agents:
class MemoryClient:
    def save_memory(self, agent_id, task_id, context, learnings):
        pass

    def get_relevant_memories(self, task_type, limit=10):
        pass

    def write_memo(self, to_agent, subject, content, priority):
        pass

    def read_memos(self, agent_id, unread_only=True):
        pass

    def search_knowledge_base(self, query, limit=5):
        pass
```

### 5. **Integration-Plan mit bestehendem System**
```markdown
# Was muss geändert werden?
- Agent Zero (main.py): Memory-Client hinzufügen
- n8n Workflows: Memory-Save-Step nach Critic
- MCP-Server: Memory-Query-Tool hinzufügen
- docker-compose.yml: pgvector-Extension aktivieren
```

### 6. **Testing-Strategy**
```markdown
# Tests:
- Unit-Tests: Memory-Save/Retrieve
- Integration-Tests: Agent → Memory → RAG
- Performance-Tests: 1000 Agents parallel
- Load-Tests: 1M Memories in DB
- Semantic-Search-Accuracy: Relevance > 0.8
```

---

## 🔍 TECHNISCHE CHALLENGES

### Challenge 1: Embedding-Performance
**Problem:** 1000 Agents fragen gleichzeitig Knowledge-Base
**Lösung-Optionen:**
- Batch-Embeddings (mehrere Queries parallel)?
- Embedding-Cache (häufige Queries)?
- Asynchrone Verarbeitung (Queue)?

### Challenge 2: Context-Window-Overflow
**Problem:** Relevante Memories > 200K Tokens
**Lösung-Optionen:**
- Summarization (LLM fasst Memories zusammen)?
- Hierarchical-Retrieval (erst grob, dann fein)?
- Dynamic-Pruning (irrelevante Memories rauswerfen)?

### Challenge 3: Memory-Staleness
**Problem:** Learning von vor 6 Monaten ist veraltet
**Lösung-Optionen:**
- Time-Decay-Scoring (alte Memories weniger relevant)?
- Manual-Review (humans markieren veraltete Memories)?
- Auto-Deprecation (nach 1 Jahr archivieren)?

### Challenge 4: Memo-Spam
**Problem:** Agent schreibt 100 Memos/Tag → alle ignorieren sie
**Lösung-Optionen:**
- Rate-Limiting (max 10 Memos/Tag per Agent)?
- Priority-Filtering (nur urgent Memos zeigen)?
- Reputation-System (Agents mit guten Memos = höhere Priorität)?

### Challenge 5: Knowledge-Base-Versioning
**Problem:** Doku ändert sich → alte Embeddings falsch
**Lösung-Optionen:**
- Auto-Reindex (bei Git-Commit)?
- Version-Tagging (Embeddings haben Version-ID)?
- Incremental-Updates (nur geänderte Docs)?

---

## 📊 REFERENZEN

### Dateien die ihr BRAUCHT:

**Must-Read (8 Dateien):**
1. `CLAUDE-MEMORY-SYSTEM.md` - Existierendes Memory-System (Basic)
2. `docs/memory/ERKENNTNISSE_KOMPLETT.md` - Learnings (groß!)
3. `docs/ground-zero-v4.1/02-Master-v4.2.md` - Architektur
4. `SKILLS-ECOSYSTEM-COMPLETE-ANALYSIS.md` - 22 Skills
5. `services/agent-zero/state_manager.py` - Checkpoints (ähnlich)
6. `infra/init/init-postgres-schema.sql` - DB-Schema
7. `docker-compose.yml` - Wo Memory-Services reinpassen
8. `.env.example` - Neue ENV-Vars

**Nice-to-Have (3):**
9. Research auf RAG-Systeme (LangChain, LlamaIndex)
10. pgvector-Dokumentation
11. PostgreSQL NOTIFY/LISTEN-Docs

---

## 🎯 KRITISCHE FRAGEN

### Architektur:
1. **Embedding-Model**: OpenAI text-embedding-3-small? Local (SBERT)?
2. **Vector-DB**: pgvector? Qdrant? Pinecone?
3. **Chunking**: 500 Tokens? 1000 Tokens? Overlap?
4. **Memory-DB**: Separate PostgreSQL? Reuse existing?

### Performance:
5. **Embedding-Latency**: < 100ms? Batch-Processing?
6. **Search-Latency**: Semantic-Search < 50ms?
7. **Concurrent-Agents**: 100? 1000? 10000?
8. **Storage-Growth**: 1M Memories = wieviel GB?

### Operations:
9. **Backup**: Memory-DB täglich? Wöchentlich?
10. **Retention**: Long-Term 1 Jahr? Unbegrenzt?
11. **Cleanup**: Automatisch? Manual-Review?
12. **Monitoring**: Memory-Hit-Rate? Query-Latency?

### Integration:
13. **Agent-API**: Python-Client? REST-API? gRPC?
14. **MCP-Tools**: memory_search, memory_save?
15. **n8n-Integration**: Memory-Node für Workflows?
16. **Skills**: Auto-Load Memory on Skill-Start?

---

## ⏱️ ZEITRAHMEN

**Projekt-Größe:** SEHR GROSS (8-10 Wochen mit 3 Leuten)

**Complexity-Level:** SEHR HOCH
- Embeddings + Vector-Search
- Multi-Layer Memory
- Performance-Optimization
- Integration mit 22 Skills

**Team-Empfehlung:**
- 1 ML-Engineer (RAG + Embeddings)
- 1 Backend-Developer (Memory-API)
- 1 Database-Engineer (PostgreSQL + pgvector)
- 1 QA (Testing + Performance)

---

## 🎯 SUCCESS CRITERIA

### Must-Have:
- [ ] 5 Memory-Ebenen implementiert
- [ ] PostgreSQL-Schema deployed
- [ ] pgvector-Extension aktiviert
- [ ] Embedding-Pipeline läuft
- [ ] Memory-API für Agents
- [ ] 4 Core Skills nutzen Memory
- [ ] Semantic-Search funktioniert (Relevance > 0.8)
- [ ] Memo-System funktioniert

### Should-Have:
- [ ] Alle 22 Skills nutzen Memory
- [ ] RAG-Pipeline optimiert (< 100ms)
- [ ] Context-Window-Management
- [ ] Auto-Cleanup für alte Memories
- [ ] Monitoring-Dashboard

### Nice-to-Have:
- [ ] Multi-Tenant Isolation
- [ ] Hierarchical-Retrieval
- [ ] Memory-Summarization
- [ ] Reputation-System für Memos

---

## 🚀 START-PUNKT

**Beginnt mit:**
1. Lest `CLAUDE-MEMORY-SYSTEM.md` (was existiert schon?)
2. Lest `ERKENNTNISSE_KOMPLETT.md` (Beispiele für Learnings)
3. Research: RAG-Systeme (LangChain, LlamaIndex, pgvector)
4. Design: High-Level Architektur (5 Ebenen)
5. Prototype: Semantic-Search mit pgvector

---

# 🎯 LOS GEHT'S - PLANT DAS MEMORY-SYSTEM!

**Das ist DAS Herzstück von intelligenten Multi-Agent-Systemen!** 🧠🚀
