# 08 – Sanctum & Agent-Zero: Der geschützte Kern

**Version:** 1.0  
**Phase:** 3 (Sanctum produktiv)  
**Säule:** C (Sanctum)  
**Datum:** 2025-11-19  

---

## 🛡️ Sanctum – Funktion und Einbettung

Sanctum ist Säule C deiner Infrastruktur: ein besonders geschützter Bereich, in dem Agent-Zero über dedizierte Queue-Tabellen mit dem Rest des Systems spricht.

Statt direkten HTTP- oder Shell-Zugriffen nutzt Agent-Zero Postgres-Tabellen wie `agent_zero_queue` und `agent_zero_results`, um Aufgaben entgegenzunehmen und Ergebnisse zurückzugeben.

Die Final-Architektur beschreibt Sanctum als Zone mit optionalem lokalem LLM, in der sensible Analysen, komplexe Korrelationen und Sicherheits-/Compliance-Bewertungen stattfinden, ohne dass Rohdaten unnötig in externe Systeme fließen.

### 🔁 Veranschaulichung – Agent-Zero-Queue-Flow

```text
[n8n / Desktop / andere Dienste]
              |
              |  schreibt Job (JSON Payload, Priorität, Deadline)
              v
      [agent_zero_queue]  (Postgres)
              |
              |  Agent-Zero pollt neue Jobs
              v
          [Agent-Zero Engine]
              |
              |  schreibt Ergebnis + Status
              v
      [agent_zero_results] (Postgres)
              |
              |  n8n/Clients lesen Ergebnis
              v
         [Reports / weitere Flows]
```

---

## 🧩 Agent-Zero-Usecases und Datenmodell (konzeptionell)

Die Projektdaten und Architektur-Docs definieren Agent-Zero als Instanz, die für komplexere Analysen, Sicherheitsbewertungen und Offline-Processing genutzt wird, z.B. Query-Analysen, Log-Auswertung oder mehrstufige Compliance-Checks.

Das Datenmodell ist bewusst über Queues realisiert, sodass du jeden Schritt auditieren kannst und die Kommunikation mit Sanctum klar nachvollziehbar bleibt.

Typische Felder der Queue-Tabellen sind konzeptionell: Identifikator, Job-Typ, Payload (z.B. JSON-Struktur), Status, Timestamps und optional Fehler-Informationen, ohne dass du bereits feste Spaltennamen in den Specs vorgibst.

### 📊 Veranschaulichung – konzeptioneller Tabellenvergleich

| Tabelle | Zweck | Inhalt (konzeptionell) |
|--------|-------|------------------------|
| `agent_zero_queue` | Eingehende Aufgaben | Job-ID, Typ, Payload, Status „pending", Zeitstempel, Priorität |
| `agent_zero_results` | Ergebnisse | Referenz auf Job-ID, Result-Payload, Status „done/failed", Fehlerinfo, Zeitstempel |

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige Sanctum-/Agent-Zero-Beschreibung |

---

*Lies als letztes `09-groundzero-MCP-Server-Design.md` für Details zum MCP-Server.*
