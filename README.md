# LCMX — Learning Content Metadata eXchange

> **Open metadata format for digital learning content, optimized for AI systems.**

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![Spec: 1.0](https://img.shields.io/badge/Spec-1.0_Draft-orange.svg)](LCMX-specification-1.0.en.md)
[![JSON Schema](https://img.shields.io/badge/JSON_Schema-2020--12-green.svg)](schema/lcmx-schema-1.0.json)

---

## The Problem

Digital learning content — whether SCORM packages, LTI tools, or standalone web courses — is a black box for external systems:

- **SCORM** only reports a learner's completion status, not *what* was taught.
- **LTI** provides a standardized launch interface but delivers zero content metadata.
- **AI agents** (recommendation engines, assistants, search systems) cannot meaningfully use the content of digital courses.

## The Solution

**LCMX** is an open, JSON-based metadata format that makes learning content machine-readable and AI-accessible — independent of the learning format or vendor.

```
lcmx.json
├── meta          ← Classical metadata (required: id, title)
├── didactics     ← Didactic metadata (target audience, learning objectives, Bloom levels)
├── structure     ← Sitemap / content tree
├── content       ← AI-prepared content summaries
└── knowledge     ← Knowledge Graph (items + relations)
```

## Key Features

| Feature | Description |
|---|---|
| **Open** | No proprietary format, no license fees |
| **Extensible** | Vendors can add custom fields under `x_*` namespace |
| **Machine-readable** | Pure JSON, validatable via JSON Schema |
| **AI-optimized** | Knowledge Graph as a first-class citizen |
| **Standards-first** | REST, JSON-LD compatible, IRI linking where applicable |
| **Backwards-compatible** | All fields except `meta.id` and `meta.title` are optional |

## Use Cases

- **AI Assistant in LMS** — Answers learner questions directly from course content via the Knowledge Graph.
- **Recommendation Engine** — Matches skill gaps with relevant courses using learning objectives and didactic profiles.
- **Search & Discovery** — Semantic search over course content without full-text access.
- **Audit & Compliance** — Verifies which courses cover which regulations (Knowledge Graph: `legal_basis` fields).
- **Course Aggregators** — Portals can describe and categorize courses from external LMS platforms.

## Repository Structure

```
├── LCMX-specification-1.0.de.md  Full specification (German)
├── LCMX-specification-1.0.en.md  Full specification (English)
├── LICENSE                       GPLv3
├── README.md                    This file
├── example/
│   └── dsgvo-course.json        Complete example: GDPR training course
└── schema/
    └── lcmx-schema-1.0.json    JSON Schema (Draft 2020-12)
```

## Quick Start

### Minimal LCMX document

Only `meta.id` and `meta.title` are required:

```json
{
  "meta": {
    "id": "urn:example:course:my-course",
    "title": "My Course"
  }
}
```

### Recommended starting point

```json
{
  "meta": {
    "id": "urn:example:course:my-course",
    "lcmx_version": "1.0",
    "title": "My Course",
    "description": "A short description of the course.",
    "duration_minutes": 30,
    "content_type": "elearning",
    "languages": ["en"]
  },
  "didactics": {
    "learning_objectives": [
      "Explain the core concepts of topic X",
      "Apply technique Y in a practical scenario"
    ],
    "bloom_levels": ["understand", "apply"]
  }
}
```

### Validate against the schema

```bash
# Using ajv-cli (Node.js)
npx ajv validate -s schema/lcmx-schema-1.0.json -d example/dsgvo-course.json

# Using check-jsonschema (Python)
pip install check-jsonschema
check-jsonschema --schemafile schema/lcmx-schema-1.0.json example/dsgvo-course.json
```

## Delivery Modes

LCMX data can be delivered in multiple ways:

| Mode | Description |
|---|---|
| **SCORM** | Place `lcmx.json` in the root of the SCORM ZIP archive |
| **LTI** | Expose a REST endpoint: `GET /lti/courses/{id}/lcmx` |
| **Generic API** | `GET /api/content/{id}/lcmx` |
| **Embedded** | Nest the LCMX object in existing API responses |

## Knowledge Graph

The Knowledge Graph is the semantically richest layer of LCMX. It represents course content as a directed, weighted network of atomic knowledge units.

**8 item types:** `concept` · `fact` · `rule` · `process` · `cause_effect` · `example` · `warning` · `question`

**30+ relation types** define how items connect — from `has_rule` and `enforced_by` to `mitigated_by` and `answered_by`.

The specification includes **AI prompts** for fully automated Knowledge Graph generation from course content. See [Section 8.4 of the specification](LCMX-specification-1.0.en.md#84-ai-generation-prompts).

## Roadmap

| Version | Planned |
|---|---|
| **1.0** (current) | Base specification, Knowledge Graph, SCORM/LTI delivery |
| **1.1** | Multilingual Knowledge Graphs, `translations` object |
| **1.2** | Competency mapping (Skills Framework, ESCO linking) |

## Contributing

Issues and Pull Requests are welcome! Whether it's a typo in the spec, a schema improvement, or a new example — all contributions help make learning content more accessible to AI systems.

## License

This project is licensed under the [GNU General Public License v3.0](LICENSE).

---

*LCMX 1.0 — Making learning content machine-readable.*
