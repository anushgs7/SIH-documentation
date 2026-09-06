# Intelligent Land Record Digitization and Validation System — Architecture Plan

Problem Statement 26018 (SIH)

---

## 1. Guiding principles

- **Async-first**: OCR/CV/NLP on scanned documents is slow and CPU/GPU-heavy. Never run it in the request/response cycle — queue it and let the UI poll or subscribe for status.
- **Confidence-driven, not accuracy-obsessed**: no model will hit 100% on faded, handwritten, multi-language legacy records. The system's real value is knowing *which* fields it's unsure about and routing only those to a human.
- **Human-in-the-loop is a first-class feature**, not an afterthought — this is what actually gets scanned records into a trustworthy database, and it's a strong demo differentiator.
- **Everything is auditable**: land records are legal documents. Every extraction, edit, approval, and rejection needs an immutable trail of who/what/when.
- **Adapter pattern for external integration**: you won't get real LRMS/DILRMP/GIS credentials during the hackathon, so build against a clean internal schema and put a thin adapter layer between it and any external system.

---

## 2. Layered architecture

```
┌─────────────────────────────────────────────────────────┐
│ 1. CLIENT LAYER                                          │
│    Upload portal · Verifier console · Admin dashboards   │
│    (React SPA; Leaflet map view for geo-linked records)  │
└───────────────────────────┬───────────────────────────────┘
┌───────────────────────────▼───────────────────────────────┐
│ 2. API GATEWAY                                             │
│    REST/GraphQL API · JWT auth · RBAC · rate limiting      │
└───────────────────────────┬───────────────────────────────┘
┌───────────────────────────▼───────────────────────────────┐
│ 3. APPLICATION SERVICES (Flask/FastAPI, modular)            │
│    Ingestion · Preprocessing · OCR · NLP extraction ·       │
│    Validation rules · Verification workflow · Audit ·       │
│    Integration adapters · Notifications · Analytics         │
└───────────────────────────┬───────────────────────────────┘
┌───────────────────────────▼───────────────────────────────┐
│ 4. ASYNC WORKER LAYER (Celery + Redis/RabbitMQ broker)      │
│    Each pipeline stage is a task; failures retry/land in    │
│    a dead-letter queue for manual inspection                │
└───────────────────────────┬───────────────────────────────┘
┌───────────────────────────▼───────────────────────────────┐
│ 5. DATA LAYER                                               │
│    PostgreSQL + PostGIS (records, geo) · Object storage     │
│    (MinIO/S3, raw scans) · Redis (cache/queue) ·             │
│    Elasticsearch (optional, full-text search)                │
└───────────────────────────┬───────────────────────────────┘
┌───────────────────────────▼───────────────────────────────┐
│ 6. EXTERNAL INTEGRATION                                      │
│    LRMS · DILRMP · GIS (GeoServer) · state government APIs   │
└───────────────────────────────────────────────────────────┘
```

Audit logging and RBAC checks cut across every layer rather than sitting in one box — treat them as middleware, not a service you call at the end.

---

## 3. Processing pipeline (the core value chain)

1. **Upload** — user uploads a scan/PDF/image via the portal. File goes straight to object storage; a DB row is created with `status=queued`. The web request returns immediately with a document ID (no waiting on ML).
2. **Preprocessing** (Celery worker) — deskew, denoise, binarize (OpenCV); detect script/language; segment the page into regions (printed text, handwritten annotations, tables, stamps/signatures) using a layout model.
3. **OCR** (Celery worker) — printed regions go through a general OCR engine; handwritten regions go through a handwriting-specialized model. Multi-language support is handled per detected script.
4. **NLP extraction** (Celery worker) — a NER model plus regex/pattern rules over the OCR text pull out the predefined fields: landowner name, survey number, khasra number, khata number, plot area, village, tehsil, district, land classification, ownership type, mutation details, registration info.
5. **Confidence scoring** — each extracted field carries a combined OCR-confidence + NER-confidence score. Above threshold → auto-accepted. Below threshold → flagged for review.
6. **Validation & business rules** — cross-check against master village/tehsil/district lists, duplicate detection (fuzzy match on survey/khata numbers), internal consistency checks (e.g. plot areas summing correctly within a survey number).
7. **Human verification** — low-confidence or failed-validation records land in a reviewer queue. The verifier UI shows the original scan next to the extracted fields, side by side, with the low-confidence fields highlighted, so corrections take seconds rather than a full re-key.
8. **Commit & geo-link** — approved records are written to the canonical table and, where survey/plot boundary data exists, linked into the PostGIS/GIS layer.
9. **Feedback loop** — every human correction is logged as labeled training data. Periodically retraining/fine-tuning the OCR and NER models on this data is what makes the system's stated "improves accuracy over time" claim real rather than aspirational.
10. **Audit** — every transition above (queued → processing → needs-review → verified → committed) is written to an append-only audit log with actor, timestamp, and before/after diff.

---

## 4. Structured field schema

Store each extracted field with its own provenance rather than flattening straight into the record:

| Column | Purpose |
|---|---|
| `raw_ocr_text` | Unmodified OCR output for the field region |
| `extracted_value` | Cleaned/normalized value after NLP extraction |
| `confidence_score` | Combined OCR + NER confidence |
| `verification_status` | auto_accepted / pending_review / verified / rejected |
| `verified_by`, `verified_at` | Reviewer identity and timestamp |
| `source_document_id` | Link back to the original scan in object storage |
| `geo_reference` | Linked plot/survey geometry, where available |

Target fields: landowner details, survey number, khasra number, khata number, plot area, village, tehsil, district, land classification, ownership details, mutation records, registration information — matching the problem statement's list.

---

## 5. Suggested tech stack

| Component | Recommendation | Why |
|---|---|---|
| Backend framework | Flask + Celery (or FastAPI if the team wants native async) | Matches your existing Flask/blueprint experience from SHRDAA and Attention Engine — fastest to build under hackathon time pressure |
| Preprocessing / CV | OpenCV, LayoutParser (or PP-Structure) | Deskew/denoise/binarize + layout segmentation, all pretrained — no training needed |
| OCR | PaddleOCR (strong multilingual support) or Tesseract with Indic trained data; TrOCR or a cloud handwriting API (Google Vision / Azure Form Recognizer) as a pragmatic fallback for handwritten text | Avoids training a custom handwriting model from scratch in 36 hours |
| NLP / NER | spaCy custom NER, or a fine-tuned multilingual transformer (IndicBERT/MuRIL), backed by regex for strongly-patterned fields (survey/khata numbers) | Regex catches the easy, high-precision cases; the model handles free-text like owner names |
| Database | PostgreSQL + PostGIS | Structured data plus native geospatial support for plot boundaries |
| Object storage | MinIO (self-hosted, S3-compatible) | Easy to run locally for a demo, drop-in compatible with S3 for production |
| Queue / async | Redis + Celery | Decouples the web tier from slow ML processing |
| Frontend | React (verifier console, admin dashboard) + Leaflet for map views | |
| Dashboards | Chart.js/Plotly embedded in the React app, or Apache Superset if time allows | |
| Auth | JWT-based sessions, role-based access control (Data Entry Operator, Verifier, District Admin, State Admin) | |
| GIS | Leaflet (frontend) + GeoServer (backend map tile/feature serving) | |
| Deployment (production framing) | NIC MeghRaj / state government cloud, containerized via Docker/Kubernetes | Matches the government-cloud expectation in the problem statement |

---

## 6. What to actually build for the 36-hour hackathon demo

Full DILRMP/GIS integration and custom-trained handwriting models are not realistic in a hackathon window. A stronger demo focuses on:

- **Working end-to-end pipeline** on 2–3 languages (e.g. Hindi + English + one regional script) rather than "all major Indian languages" — pretrained OCR engines handle this without training.
- **The human verification UI** — this is the most visually convincing and technically honest part of the system; judges respond well to seeing confidence scores driving a real review queue rather than a black-box "99% accurate" claim.
- **A believable dashboard** — documents processed, accuracy trend, pending review count, error stats — even with a modest volume of seeded sample data.
- **Stubbed integration** — a mock REST endpoint standing in for LRMS/DILRMP/GIS, with the adapter layer built so swapping in a real endpoint later is a config change, not a rewrite.
- **Audit trail visible in the UI** — a simple "history" view per record showing every state change reinforces the governance/trust angle the problem statement cares about.

---

## 7. Security & compliance notes

- Landowner names and ownership details are personal data — apply data minimization and RBAC at the field level, not just the endpoint level.
- Encrypt object storage at rest and use TLS everywhere in transit.
- Keep the audit log append-only (no updates/deletes) so it can stand as a legal record of who changed what.
- Mention data-protection alignment (India's DPDP Act) explicitly in the pitch — evaluators for a government-facing problem statement will expect it.
