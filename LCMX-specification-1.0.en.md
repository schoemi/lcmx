# LCMX 1.0 — Learning Content Metadata Exchange

> **Status:** Draft 1.0  
> **License:** [MIT](LICENSE)  
> **Maintainer:** Dirk Schoemakers
> **Contact:** dirk@schoemakers.com
> **Feedback & Contributions:** Suggestions for improvements and additions are welcome as Issues and Pull Requests. Direct communication is also welcome.

---

## Table of Contents

1. [Motivation & Problem Statement](#1-motivation--problem-statement)
2. [Goals & Design Principles](#2-goals--design-principles)
3. [Use Cases](#3-use-cases)
4. [Delivery Modes](#4-delivery-modes)
5. [Data Model](#5-data-model)
   - 5.1 [Classical Metadata (`meta`)](#51-classical-metadata-meta)
   - 5.2 [Didactic Metadata (`didactics`)](#52-didactic-metadata-didactics)
   - 5.3 [Structure Data (`structure`)](#53-structure-data-structure)
   - 5.4 [Content Data (`content`)](#54-content-data-content)
   - 5.5 [Knowledge Graph (`knowledge`)](#55-knowledge-graph-knowledge)
6. [JSON Schema Overview](#6-json-schema-overview)
7. [Complete Example `lcmx.json`](#7-complete-example-lcmxjson)
8. [Knowledge Graph — Specification](#8-knowledge-graph--specification)
   - 8.1 [Item Types](#81-item-types)
   - 8.2 [Relation Types](#82-relation-types)
   - 8.3 [Quality Rules](#83-quality-rules)
   - 8.4 [AI Generation (Prompts)](#84-ai-generation-prompts)
   - 8.5 [Example Output](#85-example-output)
9. [Implementation Notes](#9-implementation-notes)
10. [Roadmap](#10-roadmap)
11. [Glossary](#11-glossary)

---

## 1. Motivation & Problem Statement

Digital learning content — whether SCORM packages, LTI tools, or standalone web courses — is virtually opaque to external systems:

- **SCORM** only reports a learner's completion status to the LMS, not *what* was taught.
- **LTI** provides a standardized launch interface but delivers no content metadata to other systems.
- **AI agents** (recommendation engines, assistants, search systems) therefore cannot meaningfully use the content of digital courses.

**LCMX** solves this problem with an open, JSON-based metadata format that can be used independently of the learning format and vendor.

---

## 2. Goals & Design Principles

| Principle | Implementation |
|---|---|
| **Open** | No proprietary format, no license fees |
| **Extensible** | Vendors can add custom fields under the `x_*` namespace |
| **Machine-readable** | Consistently JSON, validatable via JSON Schema |
| **AI-optimized** | Knowledge Graph as a first-class citizen in the model |
| **Standards-first** | REST, JSON-LD compatible, IRI linking where appropriate |
| **Backwards-compatible** | All fields except `meta.id` and `meta.title` are optional |

---

## 3. Use Cases

- **AI Assistant in LMS:** Answers learner questions directly based on course content (Knowledge Graph).
- **Recommendation Engine:** Matches open skill gaps of a learner with suitable course content (learning objectives, didactic profile).
- **Search & Discovery:** Semantic search over course content without full-text access.
- **Audit & Compliance:** Verifies which courses cover which standards / legal requirements (Knowledge Graph: `legal_basis` fields).
- **Course Aggregators:** Portals can describe and correctly categorize courses from external LMS platforms.

---

## 4. Delivery Modes

LCMX data can be delivered in multiple ways. A content provider chooses the method appropriate for their format.

### 4.1 SCORM Package

An `lcmx.json` file is placed in the root directory of the SCORM ZIP archive.

```
course.zip
├── imsmanifest.xml
├── lcmx.json          ← LCMX metadata
├── index.html
└── ...
```

The LMS or an upstream processor can read the file during import.

### 4.2 LTI — REST Endpoint

The content provider exposes an additional HTTP endpoint:

```
GET /lti/courses/{course_id}/lcmx
Authorization: Bearer <LTI-Token>
Content-Type: application/json
```

The response is a valid LCMX document.

### 4.3 Generic API Endpoint

For content outside of SCORM/LTI (e.g., video libraries, wikis, documentation systems):

```
GET /api/content/{id}/lcmx
```

### 4.4 Embedding in Existing Metadata APIs

LCMX objects can be embedded as a property in other API responses:

```json
{
  "id": "course-42",
  "lcmx": { ... }
}
```

---

## 5. Data Model

An LCMX document is a JSON object with four optional sections and one required section.

```
lcmx.json
├── meta          Classical metadata (required: id, title)
├── didactics     Didactic metadata
├── structure     Sitemap / content tree
├── content       AI-prepared content summary
└── knowledge     Knowledge Graph (items + relations)
```

---

### 5.1 Classical Metadata (`meta`)

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Unique course identifier (URI or UUID recommended) |
| `title` | string | ✅ | Name of the training |
| `description` | string | | Short description (max. 500 characters) |
| `duration_minutes` | integer | | Nominal learning duration in minutes |
| `release_date` | string (ISO 8601) | | Publication date |
| `updated_date` | string (ISO 8601) | | Date of the last update |
| `content_type` | string (enum) | | Content type (see below) |
| `languages` | string[] | | Languages as BCP 47 tags, e.g., `["de", "en"]` |
| `version` | string | | Course version number |
| `provider` | string | | Provider / vendor name |
| `lcmx_version` | string | | LCMX specification version, e.g., `"1.0"` |

**`content_type` Enum:**

`elearning` · `video` · `test` · `quiz` · `document` · `podcast` · `webinar` · `blended` · `microlearning` · `other`

---

### 5.2 Didactic Metadata (`didactics`)

| Field | Type | Required | Description |
|---|---|---|---|
| `target_audience` | string[] | | Target audience descriptions |
| `learning_objectives` | string[] | | Learning objectives in plain text |
| `prerequisites` | string[] | | Prerequisites / recommended prior knowledge |
| `didactic_ratio` | object | | Distribution of didactic dimensions (0–1, sum = 1) |
| `didactic_ratio.knowledge_transfer` | number | | Proportion of pure knowledge transfer |
| `didactic_ratio.transfer` | number | | Proportion of transfer / application |
| `didactic_ratio.motivation` | number | | Proportion of motivation / engagement |
| `bloom_levels` | string[] | | Covered Bloom's taxonomy levels |
| `certification` | boolean | | Does the course lead to a certificate? |

---

### 5.3 Structure Data (`structure`)

The structure is represented as an ordered tree of nodes.

| Field | Type | Description |
|---|---|---|
| `nodes` | Node[] | List of all content nodes (chapters, pages, sections) |

**Node Object:**

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique node ID within the course |
| `title` | string | Node title |
| `type` | string | `chapter` · `page` · `section` · `quiz` · `video` |
| `path` | string | Relative path to the accessible page, e.g., `/chapter1/page2.html` |
| `order` | integer | Order within the parent node |
| `parent_id` | string | ID of the parent node (`null` = top-level) |
| `duration_minutes` | integer | Estimated completion time |
| `children` | Node[] | Child nodes (alternative to `parent_id`) |

---

### 5.4 Content Data (`content`)

| Field | Type | Description |
|---|---|---|
| `summary` | string | AI-generated short summary of the entire course (max. 1000 characters) |
| `key_topics` | string[] | The most important topics / keywords |
| `pages` | PageContent[] | Page-by-page prepared content for AI processing |

**PageContent Object:**

| Field | Type | Description |
|---|---|---|
| `node_id` | string | Reference to a node from `structure.nodes` |
| `ai_summary` | string | AI-generated summary of the page |
| `ai_content` | string | AI-prepared full text of the page (Markdown) |

---

### 5.5 Knowledge Graph (`knowledge`)

The Knowledge Graph is the semantically richest layer of LCMX. It describes the course content as a network of atomic knowledge units and their relationships.

→ Full specification: see [Section 8](#8-knowledge-graph--specification)

| Field | Type | Description |
|---|---|---|
| `items` | KnowledgeItem[] | List of all knowledge units |
| `relations` | KnowledgeRelation[] | List of all relationships between items |
| `generated_at` | string (ISO 8601) | Timestamp of the (most recent) generation |
| `generator` | string | Tool / model that generated the graph |
| `schema_version` | string | LCMX schema version, e.g., `"1.0"` |

---

## 6. JSON Schema Overview

The complete JSON Schema is available at `schema/lcmx-schema-1.0.json` in this repository.

**Top-Level Structure:**

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

## 7. Complete Example `lcmx.json`

```json
{
  "meta": {
    "id": "urn:example:course:dsgvo-grundlagen",
    "lcmx_version": "1.0",
    "title": "GDPR Fundamentals for Employees",
    "description": "Mandatory training on the General Data Protection Regulation for all employees.",
    "duration_minutes": 45,
    "release_date": "2024-03-01",
    "content_type": "elearning",
    "languages": ["de"],
    "provider": "Acme Learning GmbH",
    "version": "2.1.0"
  },
  "didactics": {
    "target_audience": ["All employees", "Managers"],
    "learning_objectives": [
      "Explain the key fundamental concepts of the GDPR",
      "Recognize a data breach and initiate the notification process"
    ],
    "prerequisites": ["None"],
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
        "title": "GDPR Fundamental Concepts",
        "type": "chapter",
        "order": 1,
        "parent_id": null
      },
      {
        "id": "p-01-01",
        "title": "What is Personal Data?",
        "type": "page",
        "path": "/chapter1/page1.html",
        "order": 1,
        "parent_id": "ch-01",
        "duration_minutes": 5
      },
      {
        "id": "ch-02",
        "title": "Data Breaches & Notification Obligations",
        "type": "chapter",
        "order": 2,
        "parent_id": null
      },
      {
        "id": "p-02-01",
        "title": "The 72-Hour Deadline",
        "type": "page",
        "path": "/chapter2/page1.html",
        "order": 1,
        "parent_id": "ch-02",
        "duration_minutes": 8
      }
    ]
  },
  "content": {
    "summary": "This course teaches the fundamentals of the GDPR. Key topics include the concept of personal data, the obligations of the data controller, and the correct procedure for data breaches including the 72-hour notification requirement.",
    "key_topics": ["GDPR", "personal data", "data breach", "notification obligation", "data protection officer"],
    "pages": [
      {
        "node_id": "p-01-01",
        "ai_summary": "Introduction to the concept of personal data as defined by Art. 4(1) GDPR.",
        "ai_content": "## What is Personal Data?\n\nAll information relating to an identified or identifiable natural person is considered personal data..."
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

## 8. Knowledge Graph — Specification

The Knowledge Graph (`knowledge` object) is the heart of LCMX for AI systems. It represents course content as a directed, weighted network of atomic knowledge units.

### 8.1 Item Types

Each item is an atomic knowledge unit. There are eight types:

#### `concept` — Term / Concept

Answers: *What IS X?*

For named entities with definitions: technical terms, roles, systems, frameworks, documents. Do not use for statements *about* a concept (→ `fact`).

```json
{
  "id": "cpt-01",
  "type": "concept",
  "title": "Personal Data",
  "body": "All information relating to an identified or identifiable natural person (Art. 4(1) GDPR).",
  "domain": "Data Protection",
  "aliases": ["PII", "Personally Identifiable Information"],
  "keywords": ["GDPR", "Art. 4"],
  "source_node_id": "p-01-01",
  "generated_by": "ai",
  "confidence": 0.98
}
```

**Additional fields:** `domain`, `aliases`, `keywords`, `source_node_id`

---

#### `fact` — Fact

Answers: *What is TRUE about X?*

Verifiable, context-independent statement. Cite the source when possible. Do not use for must/should statements (→ `rule`).

```json
{
  "id": "fct-01",
  "type": "fact",
  "title": "GDPR Effective Date",
  "body": "The GDPR has been directly applicable in all EU member states since May 25, 2018.",
  "source": "GDPR Art. 99(2)",
  "verified": true,
  "generated_by": "ai",
  "confidence": 0.99
}
```

**Additional fields:** `source`, `verified`

---

#### `rule` — Rule / Norm

Answers: *What MUST / SHOULD happen?*

Normative statement with obligation level. Do not use for sequential steps (→ `process`).

```json
{
  "id": "rul-01",
  "type": "rule",
  "title": "72-Hour Data Breach Notification Obligation",
  "body": "Data breaches must be reported to the competent supervisory authority without undue delay, no later than 72 hours after becoming aware of the breach.",
  "obligation": "must",
  "legal_basis": "GDPR Art. 33(1)",
  "scope": "All controllers in case of non-trivial risk incidents",
  "generated_by": "ai",
  "confidence": 0.98
}
```

**`obligation` Enum:** `must` · `should` · `may` · `must_not`

**Additional fields:** `obligation`, `legal_basis`, `scope`

---

#### `process` — Process / Workflow

Answers: *HOW does X work / proceed?*

Ordered steps with a defined trigger and outcome. `body` contains numbered Markdown steps.

```json
{
  "id": "prc-01",
  "type": "process",
  "title": "Data Breach Notification Process",
  "body": "**Trigger:** Discovery of a data breach\n\n1. Document the incident internally\n2. Inform the Data Protection Officer\n3. Conduct a risk assessment\n4. Report to the supervisory authority (< 72h)\n5. Notify affected individuals\n\n**Outcome:** Incident reported and documented",
  "trigger": "Discovery of a data breach",
  "outcome": "Compliant completion of the notification obligation",
  "steps_count": 5,
  "roles": ["Data Protection Officer", "IT Security"],
  "generated_by": "ai",
  "confidence": 0.94
}
```

**Additional fields:** `trigger`, `outcome`, `steps_count`, `roles`

---

#### `cause_effect` — Cause & Effect

Answers: *What HAPPENS when X occurs / is violated?*

Causal chain with severity level. Ideal for risks, sanctions, escalations.

```json
{
  "id": "cse-01",
  "type": "cause_effect",
  "title": "Failure to Report → Fine up to EUR 10 Million",
  "body": "If a reportable data breach is not reported within the deadline, fines of up to 10 million euros or 2% of global annual revenue may be imposed.",
  "cause": "Failure to report or late reporting of a data breach",
  "effect": "Fine up to EUR 10 million or 2% of annual revenue",
  "severity": "critical",
  "probability": "medium",
  "generated_by": "ai",
  "confidence": 0.97
}
```

**`severity` Enum:** `low` · `medium` · `high` · `critical`  
**`probability` Enum:** `low` · `medium` · `high`

---

#### `example` — Example

Answers: *What does X look like in practice?*

Specific, memorable scenario. Not a generic example. Indicate polarity.

```json
{
  "id": "exm-01",
  "type": "example",
  "title": "Data Breach: Unencrypted Laptop Stolen",
  "body": "A sales representative loses their laptop containing unencrypted customer data. The company discovers the incident on Monday — the 72-hour deadline expires Thursday at 09:00 AM.",
  "polarity": "negative",
  "realistic": true,
  "generated_by": "ai",
  "confidence": 0.88
}
```

**`polarity` Enum:** `positive` · `negative`

---

#### `warning` — Warning / Common Misconception

Answers: *What do people commonly get WRONG about X?*

Documents widespread misconceptions or anti-patterns.

```json
{
  "id": "wrn-01",
  "type": "warning",
  "title": "Misconception: Deadline Starts at Incident, Not at Awareness",
  "body": "The 72-hour deadline does not start at the time of the incident, but at the time the controller becomes aware of it.",
  "misconception": "The 72-hour deadline runs from the time of the incident",
  "severity": "high",
  "generated_by": "ai",
  "confidence": 0.97
}
```

**Additional fields:** `misconception`, `severity`

---

#### `question` — Learning Question

Answers: *What should the learner be able to answer after the course?*

Learning objective formulated as a question. Specify the Bloom's level and referenced items.

```json
{
  "id": "qst-01",
  "type": "question",
  "title": "When Does the 72-Hour Notification Obligation Begin?",
  "body": "From what point in time does the 72-hour deadline for reporting a data breach begin, and to whom must the report be made?",
  "bloom_level": "understand",
  "answered_by": ["rul-01", "wrn-01", "prc-01"],
  "difficulty": "intermediate",
  "generated_by": "ai",
  "confidence": 0.99
}
```

**`bloom_level` Enum:** `remember` · `understand` · `apply` · `analyze` · `evaluate` · `create`  
**`difficulty` Enum:** `basic` · `intermediate` · `advanced`

---

### 8.2 Relation Types

Relations are **directed**: `from_id` → `to_id`.

`weight` (0.0–1.0) indicates the strength of the relationship:

| Weight | Meaning |
|---|---|
| `1.0` | Defining / mandatory |
| `0.7–0.9` | Strong, direct |
| `0.4–0.6` | Supporting, contextual |

**Relation Object:**

```json
{
  "id": "rel-001",
  "from_id": "cpt-03",
  "to_id": "rul-01",
  "relation_type": "has_rule",
  "weight": 1.0,
  "note": "Optional explanatory text"
}
```

#### Allowed Relation Types by Source Type

**From `concept`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `has_example` | example | Is illustrated by |
| `has_rule` | rule | Is regulated by |
| `has_fact` | fact | Has this property |
| `has_warning` | warning | Common misconception about this |
| `is_part_of` | concept | Is a component of |
| `is_a` | concept | Is a specialization of |
| `related_to` | concept | Thematically adjacent (use sparingly) |
| `requires` | concept | Understanding requires |

**From `fact`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `supports` | concept, rule | Provides evidence for |
| `contradicts` | fact | Contradicts |
| `qualifies` | rule | Restricts the scope of |
| `part_of_process` | process | Is a precondition / input for |

**From `rule`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `enforced_by` | process | Is implemented by |
| `has_exception` | rule, fact | Has an exception |
| `has_consequence` | cause_effect | Violation leads to |
| `derived_from` | fact, rule | Is derived from |
| `applies_to` | concept | Applies to |

**From `process`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `requires_concept` | concept | Requires knowledge of |
| `enforced_by_rule` | rule | Is mandated by |
| `can_cause` | cause_effect | Errors can cause |
| `illustrated_by` | example | Is demonstrated by |
| `precedes` | process | Must be completed before X |

**From `cause_effect`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `mitigated_by` | rule, process | Can be prevented by |
| `illustrated_by` | example | Is demonstrated by |
| `related_to` | warning | Is related to |
| `escalates_to` | cause_effect | Can escalate to |

**From `example` and `warning`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `illustrates` | concept, rule, process, cause_effect | Illustrates |
| `contrasts_with` | example | Positive vs. negative example |
| `related_to_rule` | rule | Arises from non-compliance with |

**From `question`:**

| Relation | Target Type | Meaning |
|---|---|---|
| `answered_by` | any | Preferred: use the `answered_by` field |
| `relates_to` | concept | Thematic link |
| `requires_for_answer` | concept | Required for answering |

---

### 8.3 Quality Rules

#### Q1 — Completeness
Every main concept mentioned in the course must have at least one `concept` item. Every rule must (where applicable) be linked to a `process` (`enforced_by`) or a `cause_effect` (`has_consequence`).

#### Q2 — Granularity
One item = one atomic knowledge unit. Split compound statements into separate items.

> ❌ *"The GDPR has been in effect since 2018 and requires a DPO."*  
> ✅ Two separate `fact` items.

#### Q3 — Body Quality
`body` must be a complete, self-explanatory statement. A reader without further context must be able to understand the item. Minimum: 1 sentence. Recommended: 2–4 sentences. Markdown is permitted.

#### Q4 — Title Quality
`title` is the node label in the graph: concise noun phrase, max. 120 characters, no trailing punctuation.

> ✅ *"GDPR Data Breach Notification Deadline"*  
> ❌ *"About the notification obligation"*

#### Q5 — ID Convention

| Type | Prefix | Example |
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

#### Q6 — Relation Density
Every item must have at least one incoming or outgoing relation. Target average: 1.5–3.0 relations per item. Use `related_to` only as a last resort.

#### Q7 — Learning Questions
At least one `question` item per learning objective. Choose `bloom_level` based on cognitive demand. Populate `answered_by` with 2–5 relevant item IDs.

#### Q8 — Language & Confidence
Generate all `title` and `body` fields in the course language. Set `generated_by: "ai"`. Confidence based on source explicitness:

| Confidence | Meaning |
|---|---|
| ≥ 0.95 | Verbatim or near-verbatim from the source |
| 0.80–0.94 | Clearly implied |
| 0.60–0.79 | Reasonable inference |
| < 0.60 | Mark as uncertain in the `body` |

---

### 8.4 AI Generation (Prompts)

The Knowledge Graph can be fully AI-generated. The following prompts are part of the LCMX specification.

#### System Prompt

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

#### User Prompt

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

**Preparing `{{COURSE_CONTENT}}`:**

```
## {page.title} [{page.id}]
{page.ai_content}

## Next page ...
```

The page IDs in the headings allow the AI to correctly populate the `source_node_id` field.

---

### 8.5 Example Output

The following output shows the result of a Knowledge Graph generation for a GDPR short training (45 minutes, 8 pages). It is inserted directly as `knowledge.items` and `knowledge.relations` into the LCMX document.

<details>
<summary>Expand example JSON</summary>

```json
{
  "items": [
    {
      "id": "cpt-01", "type": "concept",
      "title": "Personal Data",
      "body": "All information relating to an identified or identifiable natural person (Art. 4(1) GDPR). Typical examples: name, email address, IP address, location data, identification numbers.",
      "source_node_id": "p-01-01",
      "domain": "Data Protection",
      "aliases": ["PII", "Personally Identifiable Information"],
      "keywords": ["GDPR", "Data Protection", "Art. 4"],
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "cpt-02", "type": "concept",
      "title": "Data Controller (GDPR)",
      "body": "A natural or legal person which determines the purposes and means of the processing of personal data (Art. 4(7) GDPR). Bears primary liability.",
      "source_node_id": "p-01-02",
      "domain": "Data Protection",
      "keywords": ["Controller", "Art. 4(7)"],
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "cpt-03", "type": "concept",
      "title": "Data Breach",
      "body": "A breach of security leading to the accidental or unlawful destruction, loss, alteration, or unauthorized disclosure of personal data (Art. 4(12) GDPR).",
      "source_node_id": "p-02-01",
      "domain": "Data Protection",
      "aliases": ["Personal Data Breach", "Security Incident"],
      "generated_by": "ai", "confidence": 0.96
    },
    {
      "id": "fct-01", "type": "fact",
      "title": "GDPR Effective Date",
      "body": "The GDPR has been directly applicable in all EU member states since May 25, 2018. It replaced Directive 95/46/EC.",
      "source": "GDPR Art. 99(2)",
      "source_node_id": "p-01-01",
      "verified": true,
      "generated_by": "ai", "confidence": 0.99
    },
    {
      "id": "fct-02", "type": "fact",
      "title": "GDPR Fine Framework",
      "body": "The GDPR provides two tiers of fines: up to EUR 10 million or 2% of global annual revenue (Art. 83(4)), or up to EUR 20 million or 4% (Art. 83(5)) for serious violations.",
      "source": "GDPR Art. 83",
      "source_node_id": "p-02-03",
      "verified": true,
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "rul-01", "type": "rule",
      "title": "72-Hour Data Breach Notification Obligation",
      "body": "Data breaches must be reported to the competent supervisory authority without undue delay, no later than 72 hours after becoming aware of the breach. The deadline starts from the moment of awareness, not from the time the incident occurred.",
      "obligation": "must",
      "legal_basis": "GDPR Art. 33(1)",
      "scope": "All controllers in case of non-trivial risk incidents",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.98
    },
    {
      "id": "rul-02", "type": "rule",
      "title": "Data Breach Documentation Obligation",
      "body": "Every data breach must be documented internally, regardless of whether a notification obligation exists. The documentation must include the circumstances, effects, and remedial actions taken.",
      "obligation": "must",
      "legal_basis": "GDPR Art. 33(5)",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.96
    },
    {
      "id": "prc-01", "type": "process",
      "title": "Data Breach Notification Process",
      "body": "**Trigger:** Discovery of a data breach\n\n1. Document the incident internally (time, type, scope)\n2. Inform the Data Protection Officer\n3. Conduct a risk assessment\n4. Report to the supervisory authority (if risk exists, < 72h)\n5. Notify affected individuals (if high risk)\n\n**Outcome:** Incident reported and documented",
      "steps_count": 5,
      "trigger": "Discovery of a data breach",
      "outcome": "Compliant completion of the notification obligation",
      "roles": ["Data Protection Officer", "IT Security", "Management"],
      "source_node_id": "p-02-02",
      "generated_by": "ai", "confidence": 0.94
    },
    {
      "id": "cse-01", "type": "cause_effect",
      "title": "Failure to Report → Fine up to EUR 10 Million",
      "body": "If a reportable data breach is not reported or not reported within the deadline to the supervisory authority, fines of up to 10 million euros or 2% of global annual revenue may be imposed.",
      "cause": "Failure to report or late reporting of a data breach",
      "effect": "Fine up to EUR 10 million or 2% of annual revenue",
      "severity": "critical",
      "probability": "medium",
      "source_node_id": "p-02-03",
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "exm-01", "type": "example",
      "title": "Data Breach: Unencrypted Laptop Stolen",
      "body": "A sales representative loses their laptop containing unencrypted customer data (name, email, revenue). The company discovers the incident on Monday. The 72-hour deadline expires Thursday at 09:00 AM.",
      "polarity": "negative",
      "realistic": true,
      "domain": "Sales / IT Security",
      "source_node_id": "p-02-02",
      "generated_by": "ai", "confidence": 0.88
    },
    {
      "id": "wrn-01", "type": "warning",
      "title": "Misconception: Deadline Starts at Incident, Not at Awareness",
      "body": "The 72-hour deadline does not start at the time of the incident, but at the time the controller becomes aware of it. If an incident is discovered late, the deadline may have already been missed.",
      "misconception": "The 72-hour deadline runs from the time of the incident",
      "severity": "high",
      "source_node_id": "p-02-01",
      "generated_by": "ai", "confidence": 0.97
    },
    {
      "id": "qst-01", "type": "question",
      "title": "When Does the 72-Hour Notification Obligation Begin?",
      "body": "From what point in time does the 72-hour deadline for reporting a data breach begin, and to whom must the report be made?",
      "bloom_level": "understand",
      "answered_by": ["rul-01", "wrn-01", "prc-01"],
      "difficulty": "intermediate",
      "generated_by": "ai", "confidence": 0.99
    },
    {
      "id": "qst-02", "type": "question",
      "title": "What Steps Should Be Taken in a Data Breach?",
      "body": "An employee reports that a laptop containing customer data has been stolen. What steps must you take next, and in what order?",
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
    { "id": "rel-004", "from_id": "cpt-03", "to_id": "cpt-01", "relation_type": "is_part_of", "weight": 0.7, "note": "A data breach always involves personal data" },
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

**Example Graph Statistics:**

| Metric | Value | Target |
|---|---|---|
| Total items | 11 | 3–5 per 10 min of course content |
| of which concept | 3 | ≥ 30% of all items |
| of which question | 2 | 1 per chapter / learning objective |
| Total relations | 16 | 1.5–3.0 per item |
| Relations per item ∅ | 1.45 | ≥ 1.5 target |
| Items without relation | 0 | 0 (mandatory) |
| Ø confidence | 0.96 | ≥ 0.85 for curated content |

---

## 9. Implementation Notes

### For Content Providers

- Start with the `meta` section — it is the only required section.
- Add `didactics.learning_objectives` before generating the Knowledge Graph — they significantly improve AI output quality.
- Use `content.pages[].ai_content` as input for Knowledge Graph generation (prompt from Section 8.4).
- Validate the document against the JSON Schema before delivery.

### For LMS / Platform Developers

- Read `lcmx.json` during SCORM import and store the data in your database.
- Expose the metadata via an internal API for AI agents.
- Implement the LTI endpoint (`GET /lti/courses/{id}/lcmx`) for LTI content.
- The `knowledge` object is directly suitable as context for RAG systems (Retrieval-Augmented Generation).

### For AI Agents

- The Knowledge Graph is optimized as context for question-answering agents.
- Use `knowledge.items[].body` as the primary knowledge source.
- `knowledge.relations` enable graph traversal for complex queries.
- `source_node_id` allows deep links into the original content.

---

## 10. Roadmap

| Version | Planned Changes |
|---|---|
| **1.0** (current) | Base specification, Knowledge Graph, SCORM/LTI delivery modes |
| **1.1** | Multilingual Knowledge Graphs, `translations` object |
| **1.2** | Competency mapping (Skills Framework, ESCO linking) |
| **2.0** | Streaming API, incremental updates, webhooks |

---

## 11. Glossary

| Term | Definition |
|---|---|
| **LCMX** | Learning Content Metadata Exchange — this format |
| **Knowledge Graph** | Network of atomic knowledge units (items) and their relationships (relations) |
| **Item** | Atomic knowledge unit in the Knowledge Graph (e.g., a concept, a rule, a process) |
| **Relation** | Directed, weighted connection between two items |
| **SCORM** | Shareable Content Object Reference Model — widely used e-learning package format |
| **LTI** | Learning Tools Interoperability — standard for integrating learning tools |
| **RAG** | Retrieval-Augmented Generation — AI technique where external knowledge is embedded in prompts |
| **Bloom's Level** | Taxonomy level according to Bloom (remember, understand, apply, analyze, evaluate, create) |
| **confidence** | Confidence value 0–1, indicates how explicitly an AI statement is supported in the source content |

---

*LCMX 1.0 — Learning Content Metadata Exchange*  
*Specification licensed under MIT · Contributions welcome*
