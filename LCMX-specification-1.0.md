# LCMX 1.0 — Learning Content Metadata Exchange

> **Status:** Draft 1.0  
> **Lizenz:** [MIT](LICENSE)  
> **Maintainer:** *[your name / organization]*  
> **Feedback & Contributions:** Issues und Pull Requests willkommen

---

## Inhaltsverzeichnis

1. [Motivation & Problemstellung](#1-motivation--problemstellung)
2. [Ziele & Design-Prinzipien](#2-ziele--design-prinzipien)
3. [Anwendungsfälle](#3-anwendungsfälle)
4. [Lieferwege (Delivery Modes)](#4-lieferwege-delivery-modes)
5. [Datenmodell](#5-datenmodell)
   - 5.1 [Klassische Metadaten (`meta`)](#51-klassische-metadaten-meta)
   - 5.2 [Didaktische Metadaten (`didactics`)](#52-didaktische-metadaten-didactics)
   - 5.3 [Strukturdaten (`structure`)](#53-strukturdaten-structure)
   - 5.4 [Inhaltsdaten (`content`)](#54-inhaltsdaten-content)
   - 5.5 [Knowledge Graph (`knowledge`)](#55-knowledge-graph-knowledge)
6. [JSON Schema Übersicht](#6-json-schema-übersicht)
7. [Vollständiges Beispiel `lcmx.json`](#7-vollständiges-beispiel-lcmxjson)
8. [Knowledge Graph — Spezifikation](#8-knowledge-graph--spezifikation)
   - 8.1 [Item-Typen](#81-item-typen)
   - 8.2 [Relationstypen](#82-relationstypen)
   - 8.3 [Qualitätsregeln](#83-qualitätsregeln)
   - 8.4 [KI-Generierung (Prompts)](#84-ki-generierung-prompts)
   - 8.5 [Beispiel-Output](#85-beispiel-output)
9. [Implementierungshinweise](#9-implementierungshinweise)
10. [Roadmap](#10-roadmap)
11. [Glossar](#11-glossar)

---

## 1. Motivation & Problemstellung

Digitale Lerninhalte — ob SCORM-Pakete, LTI-Tools oder eigenständige Web-Kurse — sind für externe Systeme nahezu undurchsichtig:

- **SCORM** teilt einem LMS lediglich den Abschlussstatus eines Lernenden mit, nicht aber *was* gelehrt wurde.
- **LTI** bietet eine standardisierte Startschnittstelle, liefert aber keinerlei inhaltliche Metadaten an andere Systeme.
- **KI-Agenten** (Empfehlungs-Engines, Assistenten, Such-Systeme) können die Inhalte digitaler Kurse daher nicht gezielt nutzen.

**LCMX** löst dieses Problem durch ein offenes, JSON-basiertes Metadaten-Format, das unabhängig vom Lernformat und vom Hersteller eingesetzt werden kann.

---

## 2. Ziele & Design-Prinzipien

| Prinzip | Umsetzung |
|---|---|
| **Offen** | Kein proprietäres Format, keine Lizenzgebühren |
| **Erweiterbar** | Hersteller können eigene Felder unter `x_*`-Namespace ergänzen |
| **Maschinenlesbar** | Konsequent JSON, validierbar per JSON Schema |
| **KI-optimiert** | Knowledge Graph als First-Class-Citizen im Modell |
| **Standards-first** | REST, JSON-LD-kompatibel, IRI-Verlinkung wo sinnvoll |
| **Rückwärtskompatibel** | Alle Felder außer `meta.id` und `meta.title` sind optional |

---

## 3. Anwendungsfälle

- **KI-Assistent im LMS:** Beantwortet Fragen eines Lernenden direkt auf Basis des Kursinhalts (Knowledge Graph).
- **Empfehlungs-Engine:** Matcht offene Kompetenzlücken eines Lernenden mit passenden Kursinhalten (Lernziele, Didaktik-Profil).
- **Suche & Discovery:** Semantische Suche über Kursinhalte ohne Volltext-Zugriff.
- **Audit & Compliance:** Überprüft, welche Kurse welche Normen / Rechtsbasis abdecken (Knowledge Graph: `legal_basis`-Felder).
- **Kurs-Aggregatoren:** Portale können Kurse eines fremden LMS inhaltlich beschreiben und korrekt kategorisieren.

---

## 4. Lieferwege (Delivery Modes)

LCMX-Daten können auf mehreren Wegen bereitgestellt werden. Ein Content-Anbieter wählt den für sein Format passenden Weg.

### 4.1 SCORM-Paket

Eine Datei `lcmx.json` wird im Root-Verzeichnis des SCORM-ZIP-Archivs abgelegt.

```
course.zip
├── imsmanifest.xml
├── lcmx.json          ← LCMX-Metadaten
├── index.html
└── ...
```

Das LMS oder ein vorgelagerter Prozessor kann die Datei beim Import auslesen.

### 4.2 LTI — REST-Endpunkt

Der Content-Provider stellt einen zusätzlichen HTTP-Endpunkt bereit:

```
GET /lti/courses/{course_id}/lcmx
Authorization: Bearer <LTI-Token>
Content-Type: application/json
```

Die Antwort ist ein valides LCMX-Dokument.

### 4.3 Generischer API-Endpunkt

Für Inhalte außerhalb von SCORM/LTI (z. B. Video-Bibliotheken, Wikis, Dokumentations-Systeme):

```
GET /api/content/{id}/lcmx
```

### 4.4 Einbettung in bestehende Metadaten-APIs

LCMX-Objekte können als Property in andere API-Antworten eingebettet werden:

```json
{
  "id": "course-42",
  "lcmx": { ... }
}
```

---

## 5. Datenmodell

Ein LCMX-Dokument ist ein JSON-Objekt mit vier optionalen Abschnitten und einer Pflichtangabe.

```
lcmx.json
├── meta          Klassische Metadaten (Pflicht: id, title)
├── didactics     Didaktische Metadaten
├── structure     Sitemap / Inhaltsbaum
├── content       KI-aufbereitete Inhaltszusammenfassung
└── knowledge     Knowledge Graph (Items + Relations)
```

---

### 5.1 Klassische Metadaten (`meta`)

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `id` | string | ✅ | Eindeutige ID des Kurses (URI oder UUID empfohlen) |
| `title` | string | ✅ | Name des Trainings |
| `description` | string | | Kurze Beschreibung (max. 500 Zeichen) |
| `duration_minutes` | integer | | Nominale Lerndauer in Minuten |
| `release_date` | string (ISO 8601) | | Veröffentlichungsdatum |
| `updated_date` | string (ISO 8601) | | Datum der letzten Aktualisierung |
| `content_type` | string (enum) | | Typ des Inhalts (siehe unten) |
| `languages` | string[] | | Sprachen als BCP 47-Tags, z. B. `["de", "en"]` |
| `version` | string | | Versionsnummer des Kurses |
| `provider` | string | | Herstellername |
| `lcmx_version` | string | | LCMX-Spezifikationsversion, z. B. `"1.0"` |

**`content_type`-Enum:**

`elearning` · `video` · `test` · `quiz` · `document` · `podcast` · `webinar` · `blended` · `microlearning` · `other`

---

### 5.2 Didaktische Metadaten (`didactics`)

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `target_audience` | string[] | | Zielgruppen-Beschreibungen |
| `learning_objectives` | string[] | | Lernziele in Klartext |
| `prerequisites` | string[] | | Voraussetzungen / empfohlene Vorkenntnisse |
| `didactic_ratio` | object | | Anteilsverteilung didaktischer Dimensionen (0–1, Summe = 1) |
| `didactic_ratio.knowledge_transfer` | number | | Anteil reine Wissensvermittlung |
| `didactic_ratio.transfer` | number | | Anteil Transfer / Anwendung |
| `didactic_ratio.motivation` | number | | Anteil Motivation / Engagement |
| `bloom_levels` | string[] | | Abgedeckte Bloom-Stufen |
| `certification` | boolean | | Führt der Kurs zu einem Zertifikat? |

---

### 5.3 Strukturdaten (`structure`)

Die Struktur wird als geordnete Baumstruktur von Knoten (`nodes`) dargestellt.

| Feld | Typ | Beschreibung |
|---|---|---|
| `nodes` | Node[] | Liste aller Inhaltsknoten (Kapitel, Seiten, Abschnitte) |

**Node-Objekt:**

| Feld | Typ | Beschreibung |
|---|---|---|
| `id` | string | Eindeutige ID des Knotens innerhalb des Kurses |
| `title` | string | Titel des Knotens |
| `type` | string | `chapter` · `page` · `section` · `quiz` · `video` |
| `path` | string | Relativer Pfad zur aufrufbaren Seite, z. B. `/chapter1/page2.html` |
| `order` | integer | Reihenfolge innerhalb des Elternknotens |
| `parent_id` | string | ID des übergeordneten Knotens (`null` = Top-Level) |
| `duration_minutes` | integer | Geschätzte Bearbeitungszeit |
| `children` | Node[] | Unterknoten (alternativ zu `parent_id`) |

---

### 5.4 Inhaltsdaten (`content`)

| Feld | Typ | Beschreibung |
|---|---|---|
| `summary` | string | KI-generierte Kurzzusammenfassung des Gesamtkurses (max. 1000 Zeichen) |
| `key_topics` | string[] | Die wichtigsten Themen / Schlagworte |
| `pages` | PageContent[] | Seitenweise aufbereiteter Inhalt für die KI-Verarbeitung |

**PageContent-Objekt:**

| Feld | Typ | Beschreibung |
|---|---|---|
| `node_id` | string | Referenz auf einen Node aus `structure.nodes` |
| `ai_summary` | string | KI-generierte Zusammenfassung der Seite |
| `ai_content` | string | Für KI aufbereiteter Volltext der Seite (Markdown) |

---

### 5.5 Knowledge Graph (`knowledge`)

Der Knowledge Graph ist die semantisch reichste Schicht von LCMX. Er beschreibt den Kursinhalt als Netz aus atomaren Wissenseinheiten und deren Beziehungen.

→ Vollständige Spezifikation: siehe [Abschnitt 8](#8-knowledge-graph--spezifikation)

| Feld | Typ | Beschreibung |
|---|---|---|
| `items` | KnowledgeItem[] | Liste aller Wissenseinheiten |
| `relations` | KnowledgeRelation[] | Liste aller Beziehungen zwischen Items |
| `generated_at` | string (ISO 8601) | Zeitpunkt der (letzten) Generierung |
| `generator` | string | Tool / Modell, das den Graphen erzeugt hat |
| `schema_version` | string | LCMX-Schemaversion, z. B. `"1.0"` |

---

## 6. JSON Schema Übersicht

Das vollständige JSON Schema ist unter `schema/lcmx-schema-1.0.json` in diesem Repository verfügbar.

**Top-Level-Struktur:**

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://lcmx.io/schema/1.0/lcmx.json",
  "title": "LCMX 1.0",
  "type": "object",
  "required": ["meta"],
  "properties": {
    "meta":       { "$ref": "#/$defs/Meta" },
    "didactics":  { "$ref": "#/$defs/Didactics" },
    "structure":  { "$ref": "#/$defs/Structure" },
    "content":    { "$ref": "#/$defs/Content" },
    "knowledge":  { "$ref": "#/$defs/Knowledge" }
  }
}
```

---

## 7. Vollständiges Beispiel `lcmx.json`

```json
{
  "meta": {
    "id": "urn:example:course:dsgvo-grundlagen",
    "lcmx_version": "1.0",
    "title": "DSGVO-Grundlagen für Mitarbeitende",
    "description": "Pflichtschulung zur Datenschutz-Grundverordnung für alle Mitarbeitenden.",
    "duration_minutes": 45,
    "release_date": "2024-03-01",
    "content_type": "elearning",
    "languages": ["de"],
    "provider": "Acme Learning GmbH",
    "version": "2.1.0"
  },
  "didactics": {
    "target_audience": ["Alle Mitarbeitenden", "Führungskräfte"],
    "learning_objectives": [
      "Die wichtigsten Grundbegriffe der DSGVO erklären können",
      "Eine Datenpanne erkennen und den Meldeprozess einleiten können"
    ],
    "prerequisites": ["Keine"],
    "didactic_ratio": {
      "knowledge_transfer": 0.5,
      "transfer": 0.35,
      "motivation": 0.15
    },
    "bloom_levels": ["remember", "understand", "apply"],
    "certification": true
  },
  "structure": {
    "nodes": [
      {
        "id": "ch-01",
        "title": "Grundbegriffe der DSGVO",
        "type": "chapter",
        "order": 1,
        "parent_id": null
      },
      {
        "id": "p-01-01",
        "title": "Was sind personenbezogene Daten?",
        "type": "page",
        "path": "/chapter1/page1.html",
        "order": 1,
        "parent_id": "ch-01",
        "duration_minutes": 5
      },
      {
        "id": "ch-02",
        "title": "Datenpannen & Meldepflichten",
        "type": "chapter",
        "order": 2,
        "parent_id": null
      },
      {
        "id": "p-02-01",
        "title": "Die 72-Stunden-Frist",
        "type": "page",
        "path": "/chapter2/page1.html",
        "order": 1,
        "parent_id": "ch-02",
        "duration_minutes": 8
      }
    ]
  },
  "content": {
    "summary": "Dieser Kurs vermittelt die Grundlagen der DSGVO. Schwerpunkte sind der Begriff der personenbezogenen Daten, die Pflichten des Verantwortlichen sowie das korrekte Vorgehen bei Datenpannen inklusive der 72-Stunden-Meldepflicht.",
    "key_topics": ["DSGVO", "personenbezogene Daten", "Datenpanne", "Meldepflicht", "Datenschutzbeauftragter"],
    "pages": [
      {
        "node_id": "p-01-01",
        "ai_summary": "Einführung in den Begriff der personenbezogenen Daten gemäß Art. 4 Nr. 1 DSGVO.",
        "ai_content": "## Was sind personenbezogene Daten?\n\nAlle Informationen, die sich auf eine identifizierte oder identifizierbare natürliche Person beziehen, gelten als personenbezogen..."
      }
    ]
  },
  "knowledge": {
    "generated_at": "2025-03-01T12:00:00Z",
    "generator": "claude-sonnet-4-20250514",
    "schema_version": "1.0",
    "items": [],
    "relations": []
  }
}
```

---

## 8. Knowledge Graph — Spezifikation

Der Knowledge Graph (`knowledge`-Objekt) ist das Herzstück von LCMX für KI-Systeme. Er repräsentiert den Kursinhalt als gerichtetes, gewichtetes Netz aus atomaren Wissenseinheiten.

### 8.1 Item-Typen

Jedes Item ist eine atomare Wissenseinheit. Es gibt acht Typen:

#### `concept` — Begriff / Konzept

Beantwortet: *Was IST X?*

Für benannte Entitäten mit Definition: Fachbegriffe, Rollen, Systeme, Frameworks, Dokumente. Nicht verwenden für Aussagen *über* ein Konzept (→ `fact`).

```json
{
  "id": "cpt-01",
  "type": "concept",
  "title": "Personenbezogene Daten",
  "body": "Alle Informationen, die sich auf eine identifizierte oder identifizierbare natürliche Person beziehen (Art. 4 Nr. 1 DSGVO).",
  "domain": "Datenschutz",
  "aliases": ["PBD", "Personal Data"],
  "keywords": ["DSGVO", "Art. 4"],
  "source_node_id": "p-01-01",
  "generated_by": "ai",
  "confidence": 0.98
}
```

**Zusätzliche Felder:** `domain`, `aliases`, `keywords`, `source_node_id`

---

#### `fact` — Faktum

Beantwortet: *Was ist WAHR über X?*

Verifizierbare, kontextunabhängige Aussage. Quelle angeben wenn möglich. Nicht verwenden für Muss/Soll-Aussagen (→ `rule`).

```json
{
  "id": "fct-01",
  "type": "fact",
  "title": "DSGVO-Geltungsbeginn",
  "body": "Die DSGVO ist seit dem 25. Mai 2018 in allen EU-Mitgliedstaaten unmittelbar anwendbar.",
  "source": "DSGVO Art. 99 Abs. 2",
  "verified": true,
  "generated_by": "ai",
  "confidence": 0.99
}
```

**Zusätzliche Felder:** `source`, `verified`

---

#### `rule` — Regel / Norm

Beantwortet: *Was MUSS / SOLL geschehen?*

Normative Aussage mit Verbindlichkeitsstufe. Nicht verwenden für sequenzielle Schritte (→ `process`).

```json
{
  "id": "rul-01",
  "type": "rule",
  "title": "72-Stunden-Meldepflicht bei Datenpannen",
  "body": "Datenpannen müssen der zuständigen Aufsichtsbehörde ohne unangemessene Verzögerung, spätestens binnen 72 Stunden nach Bekanntwerden, gemeldet werden.",
  "obligation": "must",
  "legal_basis": "DSGVO Art. 33 Abs. 1",
  "scope": "Alle Verantwortlichen bei nicht-risikolosem Vorfall",
  "generated_by": "ai",
  "confidence": 0.98
}
```

**`obligation`-Enum:** `must` · `should` · `may` · `must_not`

**Zusätzliche Felder:** `obligation`, `legal_basis`, `scope`

---

#### `process` — Prozess / Ablauf

Beantwortet: *WIE funktioniert / läuft X ab?*

Geordnete Schritte mit definiertem Auslöser und Ergebnis. `body` enthält nummerierte Markdown-Schritte.

```json
{
  "id": "prc-01",
  "type": "process",
  "title": "Meldeprozess bei Datenpanne",
  "body": "**Auslöser:** Entdeckung einer Datenpanne\n\n1. Vorfall intern dokumentieren\n2. Datenschutzbeauftragten informieren\n3. Risikobeurteilung durchführen\n4. Aufsichtsbehörde melden (< 72h)\n5. Betroffene Personen benachrichtigen\n\n**Ergebnis:** Vorfall gemeldet und dokumentiert",
  "trigger": "Entdeckung einer Datenpanne",
  "outcome": "Regelkonformer Abschluss der Meldepflicht",
  "steps_count": 5,
  "roles": ["Datenschutzbeauftragter", "IT-Sicherheit"],
  "generated_by": "ai",
  "confidence": 0.94
}
```

**Zusätzliche Felder:** `trigger`, `outcome`, `steps_count`, `roles`

---

#### `cause_effect` — Ursache-Wirkung

Beantwortet: *Was PASSIERT, wenn X eintritt / verletzt wird?*

Kausalkette mit Schweregrad. Ideal für Risiken, Sanktionen, Eskalationen.

```json
{
  "id": "cse-01",
  "type": "cause_effect",
  "title": "Unterlassene Meldung → Bußgeld bis 10 Mio. EUR",
  "body": "Wird eine meldepflichtige Datenpanne nicht fristgerecht gemeldet, drohen Bußgelder bis zu 10 Millionen Euro oder 2 % des weltweiten Jahresumsatzes.",
  "cause": "Unterlassene oder verspätete Meldung einer Datenpanne",
  "effect": "Bußgeld bis 10 Mio. EUR oder 2 % Jahresumsatz",
  "severity": "critical",
  "probability": "medium",
  "generated_by": "ai",
  "confidence": 0.97
}
```

**`severity`-Enum:** `low` · `medium` · `high` · `critical`  
**`probability`-Enum:** `low` · `medium` · `high`

---

#### `example` — Beispiel

Beantwortet: *Wie sieht X konkret aus?*

Spezifisches, einprägsames Szenario. Kein generisches Beispiel. Polarität angeben.

```json
{
  "id": "exm-01",
  "type": "example",
  "title": "Datenpanne: Unverschlüsseltes Notebook gestohlen",
  "body": "Ein Vertriebsmitarbeiter verliert sein Notebook mit unverschlüsselten Kundendaten. Das Unternehmen erkennt den Vorfall am Montag — die 72-Stunden-Frist endet Donnerstag 09:00 Uhr.",
  "polarity": "negative",
  "realistic": true,
  "generated_by": "ai",
  "confidence": 0.88
}
```

**`polarity`-Enum:** `positive` · `negative`

---

#### `warning` — Warnung / häufiger Irrtum

Beantwortet: *Was verstehen Menschen an X FALSCH?*

Dokumentiert verbreitete Missverständnisse oder Anti-Patterns.

```json
{
  "id": "wrn-01",
  "type": "warning",
  "title": "Irrtum: Frist beginnt beim Vorfall, nicht bei Kenntnis",
  "body": "Die 72-Stunden-Frist beginnt nicht mit dem Zeitpunkt des Vorfalls, sondern mit dem Zeitpunkt der Kenntnis durch den Verantwortlichen.",
  "misconception": "Die 72-Stunden-Frist läuft ab Eintrittszeitpunkt",
  "severity": "high",
  "generated_by": "ai",
  "confidence": 0.97
}
```

**Zusätzliche Felder:** `misconception`, `severity`

---

#### `question` — Lernfrage

Beantwortet: *Was soll der Lernende nach dem Kurs beantworten können?*

Lernziel als Frage formuliert. Bloom-Stufe und verweisende Items angeben.

```json
{
  "id": "qst-01",
  "type": "question",
  "title": "Wann beginnt die 72-Stunden-Meldepflicht?",
  "body": "Ab welchem Zeitpunkt beginnt die 72-Stunden-Frist für die Meldung einer Datenpanne, und an wen muss die Meldung erfolgen?",
  "bloom_level": "understand",
  "answered_by": ["rul-01", "wrn-01", "prc-01"],
  "difficulty": "intermediate",
  "generated_by": "ai",
  "confidence": 0.99
}
```

**`bloom_level`-Enum:** `remember` · `understand` · `apply` · `analyze` · `evaluate` · `create`  
**`difficulty`-Enum:** `basic` · `intermediate` · `advanced`

---

### 8.2 Relationstypen

Relationen sind **gerichtet**: `from_id` → `to_id`.

`weight` (0.0–1.0) gibt die Stärke der Beziehung an:

| Gewicht | Bedeutung |
|---|---|
| `1.0` | Definierend / verpflichtend |
| `0.7–0.9` | Stark, direkt |
| `0.4–0.6` | Unterstützend, kontextuell |

**Relation-Objekt:**

```json
{
  "id": "rel-001",
  "from_id": "cpt-03",
  "to_id": "rul-01",
  "relation_type": "has_rule",
  "weight": 1.0,
  "note": "Optionaler Erläuterungstext"
}
```

#### Erlaubte Relationstypen nach Quell-Typ

**Von `concept`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `has_example` | example | Wird illustriert durch |
| `has_rule` | rule | Wird geregelt durch |
| `has_fact` | fact | Hat diese Eigenschaft |
| `has_warning` | warning | Häufiger Irrtum dazu |
| `is_part_of` | concept | Ist Komponente von |
| `is_a` | concept | Ist Spezialisierung von |
| `related_to` | concept | Thematisch benachbart (sparsam verwenden) |
| `requires` | concept | Verständnis setzt voraus |

**Von `fact`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `supports` | concept, rule | Liefert Evidenz für |
| `contradicts` | fact | Steht im Widerspruch zu |
| `qualifies` | rule | Schränkt Geltungsbereich ein |
| `part_of_process` | process | Ist Vorbedingung / Input von |

**Von `rule`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `enforced_by` | process | Wird umgesetzt durch |
| `has_exception` | rule, fact | Hat Ausnahme |
| `has_consequence` | cause_effect | Verletzung führt zu |
| `derived_from` | fact, rule | Leitet sich ab aus |
| `applies_to` | concept | Gilt für |

**Von `process`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `requires_concept` | concept | Setzt Kenntnis voraus |
| `enforced_by_rule` | rule | Wird vorgeschrieben durch |
| `can_cause` | cause_effect | Fehler können verursachen |
| `illustrated_by` | example | Wird gezeigt durch |
| `precedes` | process | Muss vor X abgeschlossen sein |

**Von `cause_effect`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `mitigated_by` | rule, process | Kann verhindert werden durch |
| `illustrated_by` | example | Wird gezeigt durch |
| `related_to` | warning | Steht in Bezug zu |
| `escalates_to` | cause_effect | Kann eskalieren zu |

**Von `example` und `warning`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `illustrates` | concept, rule, process, cause_effect | Illustriert |
| `contrasts_with` | example | Positiv- vs. Negativbeispiel |
| `related_to_rule` | rule | Entsteht durch Missachtung von |

**Von `question`:**

| Relation | Ziel-Typ | Bedeutung |
|---|---|---|
| `answered_by` | any | Bevorzugt: `answered_by`-Feld nutzen |
| `relates_to` | concept | Thematische Verknüpfung |
| `requires_for_answer` | concept | Zum Beantworten nötig |

---

### 8.3 Qualitätsregeln

#### Q1 — Vollständigkeit
Jedes im Kurs erwähnte Hauptkonzept muss mindestens ein `concept`-Item haben. Jede Regel muss (wenn zutreffend) mit einem `process` (`enforced_by`) oder einem `cause_effect` (`has_consequence`) verknüpft sein.

#### Q2 — Granularität
Ein Item = eine atomare Wissenseinheit. Zusammengesetzte Aussagen in separate Items aufteilen.

> ❌ *„Die DSGVO gilt seit 2018 und erfordert einen DSB."*  
> ✅ Zwei separate `fact`-Items.

#### Q3 — Body-Qualität
`body` muss eine vollständige, selbsterklärende Aussage sein. Ein Leser ohne weiteren Kontext muss das Item verstehen. Minimum: 1 Satz. Empfohlen: 2–4 Sätze. Markdown ist erlaubt.

#### Q4 — Titel-Qualität
`title` ist das Knoten-Label im Graph: prägnante Nominalphrase, max. 120 Zeichen, kein abschließendes Satzzeichen.

> ✅ *„DSGVO Data Breach Notification Deadline"*  
> ❌ *„Über die Meldepflicht"*

#### Q5 — ID-Konvention

| Typ | Präfix | Beispiel |
|---|---|---|
| concept | `cpt-` | `cpt-01`, `cpt-02` |
| fact | `fct-` | `fct-01` |
| rule | `rul-` | `rul-01` |
| process | `prc-` | `prc-01` |
| cause_effect | `cse-` | `cse-01` |
| example | `exm-` | `exm-01` |
| warning | `wrn-` | `wrn-01` |
| question | `qst-` | `qst-01` |
| relation | `rel-` | `rel-001` |

#### Q6 — Relationsdichte
Jedes Item muss mindestens eine ein- oder ausgehende Relation haben. Angestrebter Durchschnitt: 1,5–3,0 Relationen pro Item. `related_to` nur als letztes Mittel.

#### Q7 — Lernfragen
Pro Lernziel mindestens ein `question`-Item. `bloom_level` nach kognitivem Anspruch wählen. `answered_by` mit 2–5 relevanten Item-IDs befüllen.

#### Q8 — Sprache & Konfidenz
Alle `title`- und `body`-Felder in der Kurssprache generieren. `generated_by: "ai"` setzen. Konfidenz nach Explizitheit der Quelle:

| Konfidenz | Bedeutung |
|---|---|
| ≥ 0.95 | Wörtlich oder nahezu wörtlich aus der Quelle |
| 0.80–0.94 | Klar impliziert |
| 0.60–0.79 | Vernünftige Inferenz |
| < 0.60 | Im `body` als unsicher kennzeichnen |

---

### 8.4 KI-Generierung (Prompts)

Der Knowledge Graph kann vollständig KI-generiert werden. Die folgenden Prompts sind Bestandteil der LCMX-Spezifikation.

#### System-Prompt

```
You are a knowledge extraction system for digital learning content.
Your task is to analyze the full text of a training course and produce
a structured knowledge graph in JSON format, strictly conforming to the
LCMX 1.0 specification (knowledge object).

You extract knowledge, you do not summarize. Every item in the output
must represent a distinct, self-contained unit of knowledge from the
course content. Do not invent facts, rules, or concepts not present
in the provided content.

The output must validate against the LCMX JSON Schema:
https://lcmx.io/schema/1.0/lcmx.json
Relevant section: #/$defs/Knowledge

Key constraints:
- Every item requires: id, type, title (max 120 chars), body
- Every relation requires: id, from_id, to_id, relation_type
- relation_type must be one of the 30 defined enum values
- item type must be one of: concept, fact, rule, process,
  cause_effect, example, warning, question

[ITEM TYPE RULES — see Section 8.1]
[RELATION RULES — see Section 8.2]
[QUALITY RULES — see Section 8.3]

OUTPUT FORMAT:
Return ONLY valid JSON. No markdown code fences. No preamble.
The response must be directly parseable with JSON.parse().

Required top-level structure:
{
  "items": [ ...KnowledgeItem[] ],
  "relations": [ ...KnowledgeRelation[] ]
}

Do NOT wrap in a 'knowledge' key.

If content is insufficient, return:
{ "items": [], "relations": [], "error": "<reason>" }
```

#### User-Prompt

```
Analyze the following training course content and generate a complete
LCMX knowledge graph for it.

Course metadata:
Title: {{COURSE_TITLE}}
Language: {{COURSE_LANGUAGE}}
Duration: {{COURSE_DURATION_MINUTES}} minutes

Guidelines:
- Target approx. 3–5 items per 10 minutes of content
- At least one question item per major chapter
- Flag inferred (not explicit) content with confidence ≤ 0.80

Course content:
---
{{COURSE_CONTENT}}
---

Generate the knowledge graph now.
```

**Aufbereitung von `{{COURSE_CONTENT}}`:**

```
## {page.title} [{page.id}]
{page.ai_content}

## Nächste Seite ...
```

Die Page-IDs in den Überschriften ermöglichen der KI, das `source_node_id`-Feld korrekt zu befüllen.

---

### 8.5 Beispiel-Output

Der folgende Output zeigt das Ergebnis einer Knowledge-Graph-Generierung für ein DSGVO-Kurztraining (45 Minuten, 8 Seiten). Er wird direkt als `knowledge.items` und `knowledge.relations` in das LCMX-Dokument eingesetzt.

<details>
<summary>Beispiel-JSON aufklappen</summary>

```json
{
  "items": [
    {
      "id": "cpt-01", "type": "concept",
      "title": "Personenbezogene Daten",
      "body": "Alle Informationen, die sich auf eine identifizierte oder identifizierbare natürliche Person beziehen (Art. 4 Nr. 1 DSGVO). Typische Beispiele: Name, E-Mail-Adresse, IP-Adresse, Standortdaten, Kennnummern.",
      "source_node_id": "p-01-01",
      "domain": "Datenschutz",
      "aliases": ["PBD", "Personal Data"],
      "keywords": ["DSGVO", "Datenschutz", "Art. 4"],
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "cpt-02", "type": "concept",
      "title": "Verantwortlicher (DSGVO)",
      "body": "Natürliche oder juristische Person, die über Zwecke und Mittel der Verarbeitung personenbezogener Daten entscheidet (Art. 4 Nr. 7 DSGVO). Trägt die primäre Haftung.",
      "source_node_id": "p-01-02",
      "domain": "Datenschutz",
      "keywords": ["Controller", "Art. 4 Nr. 7"],
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "cpt-03", "type": "concept",
      "title": "Datenpanne (Data Breach)",
      "body": "Sicherheitsverletzung, die zur unbeabsichtigten oder unrechtmäßigen Vernichtung, Verlust, Veränderung oder unbefugten Offenlegung von personenbezogenen Daten führt (Art. 4 Nr. 12 DSGVO).",
      "source_node_id": "p-02-01",
      "domain": "Datenschutz",
      "aliases": ["Data Breach", "Sicherheitsvorfall"],
      "generated_by": "ai", "confidence": 0.96
    },
    {
      "id": "fct-01", "type": "fact",
      "title": "DSGVO-Geltungsbeginn",
      "body": "Die DSGVO ist seit dem 25. Mai 2018 in allen EU-Mitgliedstaaten unmittelbar anwendbar. Sie ersetzt die Richtlinie 95/46/EG.",
      "source": "DSGVO Art. 99 Abs. 2",
      "source_node_id": "p-01-01",
      "verified": true,
      "generated_by": "ai", "confidence": 0.99
    },
    {
      "id": "fct-02", "type": "fact",
      "title": "Bußgeldrahmen DSGVO",
      "body": "Die DSGVO sieht zwei Bußgeldrahmen vor: bis zu 10 Mio. EUR bzw. 2 % des weltweiten Jahresumsatzes (Art. 83 Abs. 4) oder bis zu 20 Mio. EUR bzw. 4 % (Art. 83 Abs. 5) bei schwerwiegenden Verstößen.",
      "source": "DSGVO Art. 83",
      "source_node_id": "p-02-03",
      "verified": true,
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "rul-01", "type": "rule",
      "title": "72-Stunden-Meldepflicht bei Datenpannen",
      "body": "Datenpannen müssen der zuständigen Aufsichtsbehörde ohne unangemessene Verzögerung, spätestens binnen 72 Stunden nach Bekanntwerden, gemeldet werden. Die Frist beginnt ab Kenntnis, nicht ab Eintritt des Vorfalls.",
      "obligation": "must",
      "legal_basis": "DSGVO Art. 33 Abs. 1",
      "scope": "Alle Verantwortlichen bei nicht-risikolosem Vorfall",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "rul-02", "type": "rule",
      "title": "Dokumentationspflicht für Datenpannen",
      "body": "Jede Datenpanne muss intern dokumentiert werden, unabhängig davon, ob eine Meldepflicht besteht. Die Dokumentation muss Umstände, Auswirkungen und ergriffene Maßnahmen umfassen.",
      "obligation": "must",
      "legal_basis": "DSGVO Art. 33 Abs. 5",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.96
    },
    {
      "id": "prc-01", "type": "process",
      "title": "Meldeprozess bei Datenpanne",
      "body": "**Auslöser:** Entdeckung einer Datenpanne\n\n1. Vorfall intern dokumentieren (Zeitpunkt, Art, Umfang)\n2. Datenschutzbeauftragten informieren\n3. Risikobeurteilung durchführen\n4. Aufsichtsbehörde melden (falls Risiko gegeben, < 72h)\n5. Betroffene Personen benachrichtigen (falls hohes Risiko)\n\n**Ergebnis:** Vorfall gemeldet und dokumentiert",
      "steps_count": 5,
      "trigger": "Entdeckung einer Datenpanne",
      "outcome": "Regelkonformer Abschluss der Meldepflicht",
      "roles": ["Datenschutzbeauftragter", "IT-Sicherheit", "Geschäftsleitung"],
      "source_node_id": "p-02-02",
      "generated_by": "ai", "confidence": 0.94
    },
    {
      "id": "cse-01", "type": "cause_effect",
      "title": "Unterlassene Meldung → Bußgeld bis 10 Mio. EUR",
      "body": "Wird eine meldepflichtige Datenpanne nicht oder nicht fristgerecht der Aufsichtsbehörde gemeldet, drohen Bußgelder bis zu 10 Millionen Euro oder 2 % des weltweiten Jahresumsatzes.",
      "cause": "Unterlassene oder verspätete Meldung einer Datenpanne",
      "effect": "Bußgeld bis 10 Mio. EUR oder 2 % Jahresumsatz",
      "severity": "critical",
      "probability": "medium",
      "source_node_id": "p-02-03",
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "exm-01", "type": "example",
      "title": "Datenpanne: Unverschlüsseltes Notebook gestohlen",
      "body": "Ein Vertriebsmitarbeiter verliert sein Notebook mit unverschlüsselten Kundendaten (Name, E-Mail, Umsatz). Das Unternehmen erkennt den Vorfall am Montag. Die 72-Stunden-Frist endet Donnerstag 09:00 Uhr.",
      "polarity": "negative",
      "realistic": true,
      "domain": "Vertrieb / IT-Sicherheit",
      "source_node_id": "p-02-02",
      "generated_by": "ai", "confidence": 0.88
    },
    {
      "id": "wrn-01", "type": "warning",
      "title": "Irrtum: Frist beginnt beim Vorfall, nicht bei Kenntnis",
      "body": "Die 72-Stunden-Frist beginnt nicht mit dem Zeitpunkt des Vorfalls, sondern mit dem Zeitpunkt der Kenntnis durch den Verantwortlichen. Wird ein Vorfall erst spät entdeckt, kann die Frist bereits abgelaufen sein.",
      "misconception": "Die 72-Stunden-Frist läuft ab Eintrittszeitpunkt",
      "severity": "high",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "qst-01", "type": "question",
      "title": "Wann beginnt die 72-Stunden-Meldepflicht?",
      "body": "Ab welchem Zeitpunkt beginnt die 72-Stunden-Frist für die Meldung einer Datenpanne, und an wen muss die Meldung erfolgen?",
      "bloom_level": "understand",
      "answered_by": ["rul-01", "wrn-01", "prc-01"],
      "difficulty": "intermediate",
      "generated_by": "ai", "confidence": 0.99
    },
    {
      "id": "qst-02", "type": "question",
      "title": "Welche Schritte sind bei einer Datenpanne einzuleiten?",
      "body": "Ein Mitarbeiter meldet, dass ein Laptop mit Kundendaten gestohlen wurde. Welche Schritte müssen Sie als nächstes unternehmen und in welcher Reihenfolge?",
      "bloom_level": "apply",
      "answered_by": ["prc-01", "rul-01", "rul-02"],
      "difficulty": "intermediate",
      "generated_by": "ai", "confidence": 0.99
    }
  ],
  "relations": [
    { "id": "rel-001", "from_id": "cpt-03", "to_id": "rul-01", "relation_type": "has_rule", "weight": 1.0 },
    { "id": "rel-002", "from_id": "cpt-03", "to_id": "rul-02", "relation_type": "has_rule", "weight": 0.9 },
    { "id": "rel-003", "from_id": "cpt-01", "to_id": "fct-01", "relation_type": "has_fact", "weight": 0.8 },
    { "id": "rel-004", "from_id": "cpt-03", "to_id": "cpt-01", "relation_type": "is_part_of", "weight": 0.7, "note": "Eine Datenpanne betrifft immer personenbezogene Daten" },
    { "id": "rel-005", "from_id": "rul-01", "to_id": "prc-01", "relation_type": "enforced_by", "weight": 1.0 },
    { "id": "rel-006", "from_id": "rul-01", "to_id": "cse-01", "relation_type": "has_consequence", "weight": 1.0 },
    { "id": "rel-007", "from_id": "fct-02", "to_id": "cse-01", "relation_type": "supports", "weight": 0.95 },
    { "id": "rel-008", "from_id": "prc-01", "to_id": "exm-01", "relation_type": "illustrated_by", "weight": 0.85 },
    { "id": "rel-009", "from_id": "prc-01", "to_id": "cse-01", "relation_type": "can_cause", "weight": 0.8 },
    { "id": "rel-010", "from_id": "cse-01", "to_id": "wrn-01", "relation_type": "related_to", "weight": 0.8 },
    { "id": "rel-011", "from_id": "wrn-01", "to_id": "rul-01", "relation_type": "related_to_rule", "weight": 0.9 },
    { "id": "rel-012", "from_id": "cse-01", "to_id": "prc-01", "relation_type": "mitigated_by", "weight": 0.9 },
    { "id": "rel-013", "from_id": "qst-01", "to_id": "rul-01", "relation_type": "answered_by", "weight": 1.0 },
    { "id": "rel-014", "from_id": "qst-01", "to_id": "wrn-01", "relation_type": "answered_by", "weight": 0.9 },
    { "id": "rel-015", "from_id": "qst-02", "to_id": "prc-01", "relation_type": "answered_by", "weight": 1.0 },
    { "id": "rel-016", "from_id": "qst-02", "to_id": "rul-01", "relation_type": "answered_by", "weight": 0.85 }
  ]
}
```

</details>

**Graph-Statistiken des Beispiels:**

| Kennzahl | Wert | Zielwert |
|---|---|---|
| Gesamte Items | 11 | 3–5 pro 10 min Kursinhalt |
| davon concept | 3 | ≥ 30 % aller Items |
| davon question | 2 | 1 pro Kapitel / Lernziel |
| Gesamte Relationen | 16 | 1,5–3,0 pro Item |
| Relationen pro Item ∅ | 1,45 | ≥ 1,5 anstreben |
| Items ohne Relation | 0 | 0 (Pflicht) |
| Ø confidence | 0,96 | ≥ 0,85 bei kuratiertem Content |

---

## 9. Implementierungshinweise

### Für Content-Anbieter

- Beginne mit dem `meta`-Abschnitt — er ist der einzige Pflichtabschnitt.
- Füge `didactics.learning_objectives` hinzu, bevor du den Knowledge Graph generierst — sie verbessern die KI-Ausgabequalität erheblich.
- Nutze `content.pages[].ai_content` als Input für die Knowledge-Graph-Generierung (Prompt aus Abschnitt 8.4).
- Validiere das Dokument gegen das JSON Schema vor der Auslieferung.

### Für LMS / Plattform-Entwickler

- Lese `lcmx.json` beim SCORM-Import und speichere die Daten in deiner Datenbank.
- Exponiere die Metadaten über eine interne API für KI-Agenten.
- Implementiere den LTI-Endpunkt (`GET /lti/courses/{id}/lcmx`) für LTI-Inhalte.
- Das `knowledge`-Objekt eignet sich direkt als Kontext für RAG-Systeme (Retrieval-Augmented Generation).

### Für KI-Agenten

- Der Knowledge Graph ist als Kontext für Frage-Antwort-Agenten optimiert.
- Nutze `knowledge.items[].body` als primäre Wissensquelle.
- `knowledge.relations` ermöglichen Graph-Traversal für komplexe Anfragen.
- `source_node_id` erlaubt Deep-Links in den Originalinhalt.

---

## 10. Roadmap

| Version | Geplante Änderungen |
|---|---|
| **1.0** (aktuell) | Basisspezifikation, Knowledge Graph, SCORM/LTI-Lieferwege |
| **1.1** | Mehrsprachige Knowledge Graphs, `translations`-Objekt |
| **1.2** | Kompetenz-Mapping (Skills Framework, ESCO-Verlinkung) |
| **2.0** | Streaming-API, inkrementelle Updates, Webhooks |

---

## 11. Glossar

| Begriff | Definition |
|---|---|
| **LCMX** | Learning Content Metadata Exchange — dieses Format |
| **Knowledge Graph** | Netz aus atomaren Wissenseinheiten (Items) und ihren Beziehungen (Relations) |
| **Item** | Atomare Wissenseinheit im Knowledge Graph (z. B. ein Konzept, eine Regel, ein Prozess) |
| **Relation** | Gerichtete, gewichtete Verbindung zwischen zwei Items |
| **SCORM** | Shareable Content Object Reference Model — verbreitetes E-Learning-Paketformat |
| **LTI** | Learning Tools Interoperability — Standard für die Integration von Lernwerkzeugen |
| **RAG** | Retrieval-Augmented Generation — KI-Technik, bei der externes Wissen in Prompts eingebettet wird |
| **Bloom-Stufe** | Taxonomie-Ebene nach Bloom (remember, understand, apply, analyze, evaluate, create) |
| **confidence** | Konfidenzwert 0–1, gibt an wie explizit eine KI-Aussage im Quellinhalt belegt ist |

---

*LCMX 1.0 — Learning Content Metadata Exchange*  
*Spezifikation lizenziert unter MIT · Beiträge willkommen*
