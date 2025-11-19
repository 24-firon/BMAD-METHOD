# 02 – MCP & Desktop-Stack: Die Kommandozentrale

**Version:** 1.0  
**Phase:** 1 (MCP-First, lokal)  
**Säule:** A (Desktop/Kommandozentrale)  
**Datum:** 2025-11-19  

---

## 🧠 Rolle des Desktop-Stacks im Gesamtprojekt

Der Desktop-Stack ist der Teil von Ground-Zero, den du physisch „anfasst": dein Windows-Rechner, dein Editor, deine Terminals, dein Claude Desktop, deine lokalen Docker-Container und MCP-Server.

In der Zielarchitektur ist dieser Stack nicht nur irgendeine Dev-Umgebung, sondern die **Kommandozentrale** (Säule A), von der aus du alle anderen Säulen – Maschinenraum und Sanctum – konfigurierst, testest und steuerst.

Die Projekt-DNA macht klar, dass Ground-Zero ein dreischichtiges System ist, in dem jede Ebene eine klar umrissene Aufgabe hat: Säule A steuert und denkt, Säule B führt Workflows und Datenhaltung aus, Säule C ist der abgesicherte Analyse-Kern.

Das bedeutet praktisch: **Wenn Säule A nicht stabil, durchschaubar und reproduzierbar ist, hat der Rest des Systems keine verlässliche Führungsinstanz**, und jede Änderung an Droplets, n8n oder DR-Strategie wäre riskant und schwer nachzuvollziehen.

### 📊 Veranschaulichung: Position des Desktop-Stacks

```text
           ┌──────────────────────────────┐
           │   Säule A: Desktop           │
           │   - Windows 11               │
           │   - WSL2 (Ubuntu)            │
           │   - Docker Desktop           │
           │   - Claude Desktop + MCP     │
           └───────────────┬──────────────┘
                           │  steuert
                           v
           ┌──────────────────────────────┐
           │   Säule B: Maschinenraum     │
           │   - Droplets + Docker        │
           │   - n8n + Postgres + E2B     │
           └───────────────┬──────────────┘
                           │  liefert Jobs/Logs
                           v
           ┌──────────────────────────────┐
           │   Säule C: Sanctum           │
           │   - Agent-Zero + Queues      │
           │   - Offline-Analysen         │
           └──────────────────────────────┘
```

---

## 🧩 Konkrete Bestandteile von Säule A

Säule A ist in deinen Dateien nicht abstrakt, sondern sehr konkret beschrieben: sie besteht aus genau definierten Tools und Dateien, die zusammenspielen müssen.

- **Windows 11** ist der Host, auf dem du Editor, Git, Terminals und Claude Desktop ausführst; er stellt auch GUI-Tools und deine Dateistruktur bereit.
- **WSL2 (Ubuntu)** ist die Linux-VM, in der Docker und Python laufen; sie dient als Container-Host und als Ort für MCP-Server-Prozesse und lokale Test-Datenbanken.
- **Docker Desktop** ist die Orchestrierungsschicht auf dem Desktop, mit der du Container für Tests (Postgres-Instanzen, E2B-Test-Container, Hilfsservices) hochfährst, ohne gleich Droplets zu belasten.
- **Python 3.10+** ist bewusst als Standard-Runtime für MCP-Server, E2B-Steuer-Code und Hilfsskripte gewählt worden, um Node/npm-Komplexität aus dem Kern zu verbannen und Fehlersuche zu vereinfachen.
- **Claude Desktop mit MCP-Client** ist die Benutzerschnittstelle, über die du und spätere Agents Tools aufrufst, Code-Snippets inspizierst und Infrastruktur-Operationen einleitest – allerdings immer über klar definierte MCP-Tools, nicht über freie Shell-Kommandos.
- **Repo-Set (`projektdaten.md`, `GROUND-ZERO-FINAL-ARCHITEKTUR.md`, `ERKENNTNISSE_KOMPLETT.md`, MCP/E2B-Docs)** ist der Wissens-Layer, der definiert, was „korrektes Verhalten" für diesen Stack bedeutet; ohne diese Docs wäre es nur eine lose Tool-Sammlung.

### 📊 Veranschaulichung: Übersicht / Komponenten-Tabelle

| Ebene | Komponente | Rolle im System |
|-------|-----------|-----------------|
| Host | Windows 11 | GUI, Editor, Git, Claude Desktop |
| VM | WSL2 (Ubuntu) | Linux-Runtime für Docker & Python |
| Runtime | Python 3.10+ | MCP-Server, E2B-Steuerung |
| Orchestrierung | Docker Desktop | Lokale Container (Postgres, E2B, Tests) |
| LLM-Frontend | Claude Desktop | Chat + MCP-Client für Tools |
| Knowledge | `projektdaten.md` + Architektur/Erkenntnisse | Zielbild, Prinzipien, Metriken |

---

## 🎯 Warum MCP bei dir so zentral ist

Der MCP-Report in deinen Dateien arbeitet heraus, dass „klassische" MCP-Nutzung schnell ineffizient wird, wenn man sehr viele Tools mit komplexen Schemas hat – die Tool-Definitionen und Zwischenergebnisse blähen den Prompt-Kontext massiv auf.

Stattdessen zeigt der Report einen Ansatz, bei dem Tool-Orchestrierung in externe Systeme (MCP-Server, E2B-Sandboxes) ausgelagert wird und der LLM-Kontext vor allem zur Planung, Auswertung und Entscheidung genutzt wird, was du in Ground-Zero übernimmst.

Für dein Projekt bedeutet das: MCP ist nicht „nice to have", sondern die **einheitliche Schicht, über die alle relevanten Aktionen angestoßen werden**, egal ob es dabei um HTTP-Calls, Dateizugriffe, E2B-Runs oder später n8n-Interaktionen geht.

Dadurch kannst du Regeln, Sicherheit und Logging auf der Tool-Ebene kontrollieren, statt jedem Prompt einzeln vertrauen zu müssen, was gerade bei DR, Compliance und Multi-Tenant-Daten entscheidend ist.

### 🔁 Veranschaulichung: MCP-Architektur aus deiner Sicht

```text
[ Claude Desktop ]
      |
      |  (MCP-Request: tool + JSON-Input)
      v
[ MCP-Client ]
      |
      |  HTTP/WS over localhost:PORT
      v
[ MCP-Server (Python) ]
      |
      +--> [Filesystem]   (z.B. projektdaten.md lesen)
      |
      +--> [HTTP-APIs]    (z.B. n8n, GitHub, Status-Endpunkte)
      |
      +--> [E2B-Sandbox]  (z.B. Magic-MCP in Sandbox)
      |
      +--> [Postgres]     (z.B. Health-Checks, Queue-Jobs)
```

---

## 🔧 Konkrete Tools und Schnittstellen, die du brauchst

In deinen Dokumenten ist nicht jedes MCP-Tool schon als endgültige Spezifikation niedergelegt, aber die **natürlichen Kandidaten** ergeben sich aus den Anforderungen von Projektdaten, Architektur und E2B-Flow.

- Ein `http.get`/`http.post`-Tool, um REST-APIs wie n8n-Health, Droplet-Status-Endpoints oder GitHub-APIs abzufragen.
- Ein `fs.read`-Tool (read-only), das konfigurierte Dateien wie `projektdaten.md`, `GROUND-ZERO-FINAL-ARCHITEKTUR.md`, `docker-compose.yml` und `env.example` lesen kann, ohne unkontrolliert im Dateisystem zu wildern.
- Ein `git.status`-Tool, mit dem Agents sehen können, ob das Repo sauber ist, welche Dateien geändert wurden und ob ein geplanter Refactor überhaupt commit-bereit ist.
- Ein `e2b.run`-Tool, das eine vordefinierte E2B-Sandbox startet, ein Code-Snippet oder einen Befehl ausführt und Logs/Ergebnisse zurückgibt, ohne deinen lokalen Host zu verschmutzen.
- Später: Tools wie `pg.health` oder `queue.enqueue_agent_zero`, um Postgres-Health zu prüfen oder Jobs in die Agent-Zero-Queues zu schreiben.

### 📊 Veranschaulichung: Mini-Tool-Katalog

| Tool-Name | Input-Schema (vereinfacht) | Output | Typische Nutzung |
|----------|----------------------------|--------|------------------|
| `http.get` | `{ "url": str }` | Status, Header, Text | n8n-Health, API-Checks |
| `fs.read` | `{ "path": str }` | File-Content | Projekt-Docs lesen, Config prüfen |
| `git.status` | `{}` | Changed-Files-Liste | Vor Refactors/Commits |
| `e2b.run` | `{ "code": str, "timeout": int }` | Logs, Exit-Code | Sandbox-Tests, Scripts |

---

## 🧪 Desktop-Stack vs. E2B-Sandbox – wer macht was?

Deine E2B-Doku beschreibt ein Setup, in dem ein Magic-MCP-Server **innerhalb** einer E2B-Sandbox läuft und von Python-Code aus gesteuert wird.

Das heißt: Der Desktop-Stack selbst führt nicht alle schweren Operationen direkt auf dem Host aus, sondern nutzt E2B-Instanzen, um komplexe oder riskante Tätigkeiten (z.B. Installationen, große Code-Analysen) gekapselt zu erledigen.

Dieses Muster passt perfekt zur Drei-Säulen-Idee: Säule A plant und steuert, nimmt aber so wenig Risiko wie möglich auf den eigenen Rücken, indem sie Arbeit an Docker/E2B-Umgebungen delegiert – zuerst lokal, später im Maschinenraum.

### 🔁 Veranschaulichung: Desktop ↔ E2B-Beziehung

```text
[ Python-Client in Säule A ]
          |
          | 1) Anfrage: "Starte E2B-Sandbox mit Magic-MCP"
          v
[ E2B-API / SDK ]
          |
          | 2) Sandbox-Container wird gestartet
          v
[ E2B-Sandbox (Linux) ]
          |
          | 3) npm run start:magic-mcp
          v
[ Magic-MCP-Server in Sandbox ]
          |
          | 4) MCP-Tools stehen Desktop zur Verfügung
          v
[ Claude Desktop nutzt Tools aus Sandbox ]
```

---

## 🎛️ Konfiguration: Ports, Env, Dateien im MCP-/Desktop-Kontext

Die Final-Architektur und E2B/MCP-Docs geben zusammen ein klares Bild, welche Konfigurationen im Desktop-Umfeld wichtig sind.

Wesentliche Punkte:

- Ein MCP-Port (z.B. `8010`) muss in WSL2 frei und erreichbar sein, damit Claude Desktop den Server ansprechen kann.
- Lokale `.env.local`-Dateien speichern MCP-bezogene Settings (`MCP_PORT`, `LOG_LEVEL`, evtl. `MCP_TOKEN`), sind aber **nicht im Repo**; Struktur wird höchstens in einem `env.example` skizziert.
- Die Pfade zu wichtigen Projekt-Docs (`projektdaten.md`, `GROUND-ZERO-FINAL-ARCHITEKTUR.md`, `ERKENNTNISSE_KOMPLETT.md`) sollten für MCP-Tools klar und stabil sein, damit Agents sie immer wieder zuverlässig lesen können.

### 📋 Veranschaulichung: Beispiel-Env für MCP-Server

```text
MCP_PORT=8010
LOG_LEVEL=INFO
MCP_TOKEN=local-dev-token-optional
```

ASCII-Mini-Topologie im Desktop-Netz:

```text
Windows Host
  ├─ Claude Desktop
  └─ WSL2 (Ubuntu)
        ├─ MCP-Server :8010
        └─ Docker (E2B-Sandbox-Images)
```

---

## 🧠 Aus deiner Sicht kritische Qualitätsmerkmale für diesen Layer

Deine Erkenntnis- und Delivery-Docs sagen sinngemäß: **„Klarheit, Meta-Infos, mehrere Perspektiven, keine verkürzenden Halbsätze"** – das gilt besonders hier, weil dieser Layer die restliche Architektur „bedient".

Konkret bedeutet das:

- Jeder wichtige MCP-/Desktop-Baustein muss in mindestens einem **Architektur-Doc** (z.B. Final-Architektur), in einem **Projekt-Doc** (`projektdaten.md`) und idealerweise in einer **Setup-Doku** (E2B/MCP-Doku) erwähnt werden, damit Agents Korrelationen erkennen können.
- Es sollte klar sein, **warum** Python-MCP und E2B gewählt wurden (Token-Effizienz, Sicherheit, Reproduzierbarkeit) und nicht nur, **dass** sie existieren.
- Meta-Perspektive: Der Desktop-Layer wird sowohl von dir als Mensch als auch von Agents benutzt; Texte müssen explizit sagen, wie beide Typen Benutzer damit interagieren sollen, um Missverständnisse zu vermeiden.

### 📊 Veranschaulichung: Meta-Matrix „Wer nutzt was?"

| Element | Mensch (du) | KI-Agent |
|---------|-------------|---------|
| `projektdaten.md` | Lesen, pflegen, Entscheidungen treffen | Lesen, Kontext extrahieren, Zitate ziehen |
| MCP-Server | Starten, Logs prüfen | Tools nutzen (`http.get`, `fs.read`, `e2b.run`) |
| E2B-Sandbox | Konzeption verstehen, gelegentlich manuell triggern | Über MCP/E2B-Tools indirekt nutzen |
| Docker Desktop | Container-Status beobachten, Images pflegen | Nur über Tools/Playbooks, keine direkten Execs |

---

## 🔗 Was für die weiteren Kapitel wichtig ist (Anschlussfähigkeit)

Ein sauber formulierter MCP-/Desktop-Block wie dieser ist nicht „nur" Beschreibung, sondern das Bindeglied zu allen anderen Teilen der Projekt-DNA.

- Für den **Maschinenraum-Block** ist entscheidend, dass klar ist, wie n8n, Postgres und Agent-Zero aus Sicht des Desktop-Stacks getestet, konfiguriert und überwacht werden (MCP-Tools → Health-Checks → Compose-Status).
- Für den **Backup/DR-Block** ist wichtig, dass die Desktop-Seite Backups anstoßen, Validierungen abfragen und Game-Day-Szenarien dokumentieren kann, ohne direkt auf dem Droplet herumzueditieren.
- Für **Compliance & Monitoring** bildet dieser Layer den Einstiegspunkt, über den Logs gelesen, Retention-Policies überprüft und n8n-Workflows inspiziert werden.

Damit ist der MCP-/Desktop-Teil jetzt in einer Form beschrieben, die sowohl **ausführlich** als auch **scan-bar** ist, mit jeweils einem klaren Textblock pro Unterthema plus der dazu passenden Visualisierung.

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige MCP-/Desktop-Beschreibung |

---

*Lies als nächstes `03-Maschinenraum-und-Compose.md` für Details zum produktiven Stack.*
