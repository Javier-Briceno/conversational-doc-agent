# Conversational Document Intelligence Agent

> 🇩🇪 Deutsche Version unten. 🇬🇧 English version collapsed below.

---

## Beschreibung

Ein konversationeller KI-Agent, der Geschäftsdokumente (PDFs) über eine IDP-Pipeline verarbeitet und es Nutzern ermöglicht, diese in natürlicher Sprache zu befragen. Der Agent unterstützt Multi-Document RAG, mehrsprachige Anfragen, Sitzungsgedächtnis und gibt ehrliche Antworten bei Fragen außerhalb des Dokumentenumfangs.

Entwickelt als Portfolio-Projekt zur Demonstration von Kompetenzen in IDP, Conversational AI und Geschäftsprozessautomatisierung.

---

## Demo — Testergebnisse

Der Agent wurde anhand von 5 Schwierigkeitsstufen mit dem Weidmüller Konzernabschluss 2021 validiert:

| Stufe | Testtyp | Ergebnis |
|---|---|---|
| 1 | Einfache Faktenabfrage | ✅ Bestanden |
| 2 | Spezifische Finanzkennzahlen | ✅ Bestanden |
| 3 | Mehrstufige Fragen mit Gedächtnis | ✅ Bestanden |
| 4 | Ehrlichkeit — Fragen außerhalb des Dokumentenumfangs | ✅ Bestanden |
| 5 | Sprachwechsel (Deutsch / Englisch) | ✅ Bestanden |

Der Agent zitierte korrekt rechtliche Ausnahmen (§286 Abs. 4 HGB) bei Fragen zu Einzelgehältern, demonstrierte sitzungsübergreifendes Gedächtnis ohne erneute Unternehmensnennung und wechselte je nach Kontext zwischen den Unternehmens-Vektorspeichern.

---

## Architektur

```
[Nutzer via Chat]
        ↓
[n8n — When chat message received]
        ↓
  Ist es ein Dokument?
   /              \
 JA               NEIN
  ↓                ↓
[Text Extractor    [AI Agent — Claude]
 from PDFs]             ↓
  ↓              [Postgres Chat Memory]
[Default                ↓
 Data Loader]    ┌─────────────────────────────┐
  ↓              │  Tool: Weidmüller           │ → filter: weidmuller_konzern_2021.pdf
[Recursive       │  Tool: Kärcher              │ → filter: kaercher_vertriebs.pdf
 Character       │  Tool: CLAAS                │ → filter: claas_jahresabschluss.pdf
 Text Splitter   └─────────────────────────────┘
 chunk=1000             ↓
 overlap=150]    [Antwort in natürlicher Sprache]
  ↓
[Embeddings OpenAI
 text-embedding-3-small]
  ↓
[Postgres PGVector Store
 Tabelle: n8n_vectors
 Metadaten: filename]
```

### Multi-Document RAG Design

Jedes Unternehmen verfügt über einen eigenen PGVector-Retrieve-Knoten mit einem festen Metadaten-Filter. Dies verhindert, dass der Agent Daten verschiedener Unternehmen vermischt. Der Agent wählt den richtigen Knoten basierend auf dem im Gespräch genannten oder aus dem Kontext abgeleiteten Unternehmen.

```
Weidmüller PGVector Store  → {"filename": "weidmuller_konzern_2021.pdf"}
Kärcher PGVector Store     → {"filename": "kaercher_vertriebs.pdf"}
CLAAS PGVector Store       → {"filename": "claas_jahresabschluss.pdf"}
```

---

## Tech Stack

| Komponente | Werkzeug |
|---|---|
| Workflow-Orchestrierung | n8n |
| Chat-Interface | n8n Built-in Chat / Telegram Bot API |
| PDF-Textextraktion | Text Extractor — [siehe Begleitprojekt](https://github.com/Javier-Briceno/PDF-Text-Extractor-Workflow) |
| Textsplitting | Recursive Character Text Splitter (chunk=1000, overlap=150) |
| Embeddings | OpenAI text-embedding-3-small |
| Vektordatenbank | PostgreSQL + pgvector |
| LLM | Anthropic Claude (n8n Anthropic Chat Model) |
| Gesprächsgedächtnis | PostgreSQL (Postgres Chat Memory) |
| Containerisierung | Docker + Docker Compose |

---

## Projektstruktur

```
conversational-doc-agent/
│
├── docs/
│   └── sample/              ← Test-PDFs (nicht enthalten, siehe unten)
│
├── n8n/
│   └── workflows/           ← n8n Workflow JSON-Exporte
│
├── db/
│   └── schema.sql           ← PostgreSQL Schema
│
├── .gitignore
└── README.md
```

---

## Testdokumente

Die PDFs sind nicht im Repository enthalten. Es handelt sich um öffentliche Dokumente, die über den Bundesanzeiger verfügbar sind.

| Erwarteter Dateiname | Unternehmen | Bezugsquelle |
|---|---|---|
| `weidmuller_konzern_2021.pdf` | WEIDMÜLLER Holding AG & Co. KG | "Weidmüller" suchen → Konzernabschluss 18.11.2022 |
| `claas_jahresabschluss.pdf` | CLAAS Selbstfahrende Erntemaschinen GmbH | "CLAAS Selbstfahrende Erntemaschinen" suchen |
| `kaercher_vertriebs.pdf` | Alfred Kärcher Vertriebs-GmbH | "Alfred Kärcher Vertriebs-GmbH", Winnenden, HRB 263370 suchen |

Speichern unter: `docs/sample/`

---

## Voraussetzungen

Keine `.env`-Datei erforderlich. Alle Zugangsdaten werden über den integrierten n8n Credential Store verwaltet. Vor der Ausführung des Workflows folgende Credentials in n8n konfigurieren:

- **Anthropic** — API-Schlüssel für das Claude Chat-Modell
- **OpenAI** — API-Schlüssel für text-embedding-3-small (nur Embeddings)
- **PostgreSQL** — Verbindungsdaten für Vektorspeicher und Chat-Gedächtnis

---

## Setup

```bash
# 1. Repository klonen
git clone https://github.com/Javier-Briceno/conversational-doc-agent.git
cd conversational-doc-agent

# 2. n8n öffnen und Workflow importieren
# http://localhost:5678
# Importieren: n8n/workflows/conversational-doc-agent.json

# 3. PDF über das Chat-Interface hochladen — die IDP-Pipeline startet automatisch

# 4. Fragen stellen
```

---

## Bekannte Einschränkungen

- **Rate Limits:** Jede Nutzeranfrage löst mehrere API-Aufrufe aus (Retrieval + LLM). `top_k` auf 3–4 Chunks pro Retrieve-Knoten begrenzen, um Token-pro-Minute-Limits zu vermeiden.
- **Neues Dokument hinzufügen:** Erfordert einen neuen PGVector-Retrieve-Knoten mit dem entsprechenden Metadaten-Filter. Der Dateiname der hochgeladenen PDF muss exakt im Filter eingetragen werden — z.B. wenn die Datei als `bosch_annual_report_2023.pdf` hochgeladen wurde, muss der Filter `{"filename": "bosch_annual_report_2023.pdf"}` lauten. Groß-/Kleinschreibung wird beachtet.
- **Keine Authentifizierung** im Chat-Interface im lokalen Betrieb.

---

<details>
<summary>🇬🇧 English Version</summary>

<br>

## Description

A conversational AI agent that processes business documents (PDFs) through an IDP pipeline and allows users to query them in natural language. The agent supports multi-document RAG, multilingual queries, session memory, and honest handling of questions outside the document scope.

Built as a portfolio project demonstrating competence in IDP, Conversational AI, and business process automation.

---

## Demo — Test Results

The agent was validated across 5 difficulty levels using the Weidmüller Konzernabschluss 2021:

| Level | Test type | Result |
|---|---|---|
| 1 | Basic factual retrieval | ✅ Passed |
| 2 | Specific financial figures | ✅ Passed |
| 3 | Multi-turn memory (chained questions) | ✅ Passed |
| 4 | Honesty — questions outside document scope | ✅ Passed |
| 5 | Language switching (German / English) | ✅ Passed |

The agent correctly cited legal exemptions (§286 Abs. 4 HGB) when asked about individual CEO salaries, demonstrated cross-turn memory without re-stating the company name, and switched between company vector stores based on context.

---

## Architecture

```
[User via Chat]
        ↓
[n8n — When chat message received]
        ↓
  Is a Document?
   /           \
 YES            NO
  ↓              ↓
[Text Extractor    [AI Agent — Claude]
 from PDFs]             ↓
  ↓              [Postgres Chat Memory]
[Default                ↓
 Data Loader]    ┌───────────────────────────┐
  ↓              │  Tool: Weidmüller         │ → filter: weidmuller_konzern_2021.pdf
[Recursive       │  Tool: Kärcher            │ → filter: kaercher_vertriebs.pdf
 Character       │  Tool: CLAAS              │ → filter: claas_jahresabschluss.pdf
 Text Splitter   └───────────────────────────┘
 chunk=1000             ↓
 overlap=150]    [Response in natural language]
  ↓
[Embeddings OpenAI
 text-embedding-3-small]
  ↓
[Postgres PGVector Store
 table: n8n_vectors
 metadata: filename]
```

### Multi-document RAG design

Each company has a dedicated PGVector Retrieve node with a fixed metadata filter. This prevents the agent from mixing data across companies. The agent selects the correct node based on the company mentioned in the query — or inferred from conversation history.

```
Weidmüller PGVector Store  → {"filename": "weidmuller_konzern_2021.pdf"}
Kärcher PGVector Store     → {"filename": "kaercher_vertriebs.pdf"}
CLAAS PGVector Store       → {"filename": "claas_jahresabschluss.pdf"}
```

---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow orchestration | n8n |
| Chat interface | n8n built-in chat / Telegram Bot API |
| PDF text extraction | Text Extractor — [see companion project](https://github.com/Javier-Briceno/PDF-Text-Extractor-Workflow) |
| Text splitting | Recursive Character Text Splitter (chunk=1000, overlap=150) |
| Embeddings | OpenAI text-embedding-3-small |
| Vector database | PostgreSQL + pgvector |
| LLM | Anthropic Claude (via n8n Anthropic Chat Model node) |
| Conversation memory | PostgreSQL (Postgres Chat Memory node) |
| Containerization | Docker + Docker Compose |

---

## Project Structure

```
conversational-doc-agent/
│
├── docs/
│   └── sample/              ← PDF test documents (not included, see below)
│
├── n8n/
│   └── workflows/           ← n8n workflow JSON exports
│
├── db/
│   └── schema.sql           ← PostgreSQL schema
│
├── .gitignore
└── README.md
```

---

## Test Documents

PDFs are not included in this repository. They are public documents available on Bundesanzeiger.

| Expected filename | Company | How to find it |
|---|---|---|
| `weidmuller_konzern_2021.pdf` | WEIDMÜLLER Holding AG & Co. KG | Search "Weidmüller" → Konzernabschluss 18.11.2022 |
| `claas_jahresabschluss.pdf` | CLAAS Selbstfahrende Erntemaschinen GmbH | Search "CLAAS Selbstfahrende Erntemaschinen" |
| `kaercher_vertriebs.pdf` | Alfred Kärcher Vertriebs-GmbH | Search "Alfred Kärcher Vertriebs-GmbH", Winnenden, HRB 263370 |

Save to: `docs/sample/`

---

## Prerequisites

No `.env` file is needed. All credentials are managed through n8n's built-in credential store. Before running the workflow, configure the following inside n8n:

- **Anthropic** — API key for the Claude chat model
- **OpenAI** — API key for text-embedding-3-small (embeddings only)
- **PostgreSQL** — connection credentials for the vector store and chat memory

---

## Setup

```bash
# 1. Clone the repository
git clone https://github.com/Javier-Briceno/conversational-doc-agent.git
cd conversational-doc-agent

# 2. Open n8n and import the workflow
# http://localhost:5678
# Import: n8n/workflows/conversational-doc-agent.json

# 3. Upload a PDF via the chat interface — the IDP pipeline runs automatically

# 4. Start asking questions
```

---

## Known Limitations

- **Rate limits:** Each user query triggers multiple API calls (retrieval + LLM). Keep `top_k` at 3–4 chunks per retrieve node to avoid hitting token-per-minute limits.
- **Adding a new document:** Requires a new PGVector Retrieve node with a matching metadata filter. The exact filename of the uploaded PDF must be manually pasted into the filter — e.g. if the file was uploaded as `bosch_annual_report_2023.pdf`, the filter must be set to `{"filename": "bosch_annual_report_2023.pdf"}`. A mismatch in casing or spelling will cause the node to return no results.
- **No authentication** on the chat interface in local mode.

</details>