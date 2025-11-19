# 07 – n8n-Workflows & Automation-Layer

**Version:** 1.0  
**Phase:** 2 (Workflows produktiv)  
**Säule:** B (Maschinenraum)  
**Datum:** 2025-11-19  

---

## 🎯 n8n als Orchestrierungs-Layer

n8n ist in Ground-Zero der zentrale Orchestrierungs-Layer im Maschinenraum: Es verbindet GitHub, Webhooks, Cron-Jobs, Monitoring-Ereignisse, GDPR-Anfragen und interne Skripte mit Postgres als Datenpool und Agent-Zero-Queues.

Die Launch- und Phase-Dokumente beschreiben dazu explizit eine n8n-Workflow-Bibliothek mit 8–12 Kern-Workflows, GDPR-Runbooks und SLA-Monitoring-Flows, die in eigenen Markdown-Dateien dokumentiert werden sollen.

### 🔁 Veranschaulichung – High-Level-Flow mit n8n

```text
[GitHub / Webhooks / Cron / Users]
                 |
                 v
             [n8n Flows]
        +------+------+--------------------------+
        |      |      |                          |
        v      v      v                          v
 [Postgres] [Backup] [Monitoring/Alerts]   [GDPR/SLA-Workflows]
        |
        v
 [Agent-Zero Queues + Reports]
```

---

## 📚 Standard-Workflow-Bibliothek (n8n WORKFLOW-LIBRARY)

In `Ground-Zero-launch-Vorbereitungen-Dev-Coach-1/2` ist für Instanz 7 eine Datei `docs/n8n/WORKFLOW-LIBRARY.md` spezifiziert, die eine katalogisierte Bibliothek von Standard-Workflows enthält.

Jeder Workflow soll dort mit Name, Beschreibung, Kategorie, Trigger, Kern-Nodes, Beispiel-Konfiguration, Node-Liste und benötigten Umgebungsvariablen beschrieben werden, ohne dass komplette n8n-JSONs im Doku-Repo landen.

Kernkategorien laut Prompt: Monitoring, Backups, GitHub-Automation, Compliance (GDPR), SLA-Reports sowie technische Error-Handler für fehlgeschlagene Workflows.

### 📊 Veranschaulichung – Auszug aus der geplanten Workflow-Bibliothek

| Workflow | Kategorie | Trigger | Kern-Nodes | Kernzweck |
|----------|-----------|---------|-----------|-----------|
| GitHub Issue → Postgres Log | GitHub-Automation | GitHub-Webhook (Issue-Event) | GitHub, Postgres | Tickets/Ereignisse aus GitHub in zentrale DB schreiben |
| Daily Droplet Healthcheck | Monitoring | Cron (täglich) | HTTP Request, Postgres, Email/Webhook | Droplet- und n8n/Postgres-Health prüfen, Alerts erzeugen |
| DB-Backup-Trigger | Backups | Cron oder manueller Trigger | Execute Command / HTTP zu Backup-Script, Postgres | `pg-backup.sh` anstoßen und Status loggen |
| Workflow Error Handler | Reliability | n8n-Error-Trigger | Postgres, Email/Webhook | Fehlgeschlagene Workflows erfassen, melden, optional neu einplanen |
| GDPR Data Export | Compliance | Webhook/Ticket | Postgres, Files, Email | Datenexport nach Art. 15 DSGVO orchestrieren |
| GDPR Delete/Anonymize | Compliance | Webhook/Ticket | Postgres | Lösch-/Anonymisierungsprozesse nach Art. 17 steuern |
| Daily SLA Report | SLA | Cron (täglich) | Postgres, Email | SLA-Metriken aggregieren und Bericht senden |
| SLA Breach Notifier | SLA | Event/Timer | HTTP/DB, Webhook/Email | SLA-Verletzungen erkennen und eskalieren |

---

## 🔐 GDPR-Runbooks über n8n

Für GDPR sind zwei dedizierte Runbooks vorgesehen: `docs/compliance/GDPR-DATA-SUBJECT-RUNBOOK.md` und `docs/compliance/GDPR-LOGGING-AND-RETENTION.md`.

Sie sollen die Prozesse für Auskunft (Art. 15) und Löschung (Art. 17) sowie Log-/Daten-Retention strukturiert beschreiben und mit n8n-Workflows und SQL-Snippets unterlegen.

Im Data-Subject-Runbook ist der High-Level-Prozess klar skizziert: Eingang eines Requests, Identifikation der Person, Suche in DB/Logs, Export-Flow, Lösch-/Anonymisierungs-Flow und lückenlose Dokumentation in Audit-Logs und n8n-Execution-Logs.

### 🔁 Veranschaulichung – GDPR Auskunft/Löschung als Prozess

```text
User-Request (Ticket/E-Mail/Webhook)
        |
        v
 [n8n Workflow "gdpr-data-subject"]
        |
        +--> Identifikation (ID/E-Mail verifizieren)
        |
        +--> Suche in Postgres & relevanten Logs
        |
        +--> Export-Flow (JSON/CSV-Bundling)
        |
        +--> Lösch-/Anonymisierungs-Flow (DB-Operationen)
        |
        +--> Audit-Eintrag (gdpr_audit_log + n8n-Executions)
```

Im Logging/Retention-Runbook definierst du Tabellen mit Datentyp, System, Aufbewahrungsfrist und Strategie (Anonymisieren vs. Löschen) und beschreibst Cron-Workflows, die diese Regeln technisch umsetzen.

---

## 📈 SLA- und Monitoring-Runbook (SLA-AND-MONITORING-RUNBOOK)

Für SLAs soll `docs/compliance/SLA-AND-MONITORING-RUNBOOK.md` entstehen, das Monitoring-Basics, Metriken und n8n-Workflows zu einem konsistenten SLA-Bild verbindet.

Es soll beschreiben, welche Checks aus `monitoring-basics` bereits existieren (Healthchecks, Disk, RAM, CPU, Container-Status) und wie daraus Business- und Technik-SLAs abgeleitet werden.

Die Datei soll außerdem konkrete Ideen für SLA-Workflows liefern: einen Breach-Notifier, einen täglichen SLA-Report und einen „Watchdog", der bei kritischen Checks sofort eskaliert, inklusive Beispiel-SQLs zur Auswertung aus einer Ereignis- oder Logs-Tabelle.

### 📊 Veranschaulichung – SLA-Workflow-Typen

| Workflow | Quelle | Aufgabe |
|----------|--------|---------|
| SLA-Breach-Notifier | Monitoring-Events/DB-Tabellen | SLA-Verletzungen erkennen, Alerts versenden, Tickets anlegen |
| Daily SLA Report | Postgres-Events/Logs | Täglichen Bericht über Uptime, Fehler, Reaktionszeiten erzeugen |
| Watchdog | Healthcheck-Fehler | Bei kritischen Checks (z.B. Postgres down) sofortige Eskalation triggern |

---

## 📝 Änderungs-Historie

| Datum | Version | Änderung |
|-------|---------|----------|
| 2025-11-19 | 1.0 | Initial – Vollständige n8n-/Automation-Beschreibung |

---

*Lies als nächstes `08-Sanctum-und-Agent-Zero.md` für Details zum Queue-Modell.*
