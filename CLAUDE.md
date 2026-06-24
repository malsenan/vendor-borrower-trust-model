# Claude instructions for this project

## What this project is now

Lumora is a credit-access platform for informal Brazilian micro-entrepreneurs (mostly women). The project has pivoted from a rules-based scorecard to a three-deliverable pipeline:

1. **WhatsApp chatbot** — intake tool that contacts leads, guides them through a conversation flow, and collects documents, photos, and text
2. **Business portfolio builder** — AI pipeline that turns raw WhatsApp uploads into a structured business profile per applicant
3. **Scoring integration** — parses the portfolio into features and calls external scoring APIs (Serasa, Belvo)

The old spec files (`Lumora_Spec_MD.md`, `Lumora_Trust_Scoring_Model_Spec_v0.md.docx`) and the rules-based scorecard in `scoring_engine/` are **no longer the source of truth**. Do not treat them as authoritative for new work.

## Environment

Always use the `.venv` at the project root. Never install packages on system Python.

```bash
# Install
.venv/bin/pip install -e ".[api,dev]"

# Test
.venv/bin/pytest tests/ -v

# Run API
.venv/bin/uvicorn api.main:app --reload
```

## Project structure (current state)

```
scoring_engine/   — old rules-based scorecard (Phase 0, largely superseded)
api/              — FastAPI app; will expand to host WhatsApp webhook and portfolio endpoints
tests/            — existing pytest suite (153 tests against old scorecard)
whatsapp/         — (to be created) webhook handler, message router, conversation state
portfolio/        — (to be created) AI pipeline: raw uploads → structured business profile JSON
```

## Lead data

Two CSV files in the repo root are the source of real leads:
- `Negativados_Leads_2026-06-17_2026-06-20.csv` — leads with negative credit history
- `Taxa_Leads_2026-06-17_2026-06-20.csv` — leads concerned about high interest rates

Key fields: `full_name`, `whatsapp_number`, `email`, `date_of_birth`, `qual_negócio_você_tem` (business type), plus survey answers about credit needs.

The pilot targets ~20 of the ~200 leads.

## Hard constraints (LGPD)

Demographics — gender, location, residential moves — are collected only for bias monitoring and display. They must never be passed to a scoring function as features. This applies to the new portfolio builder and scoring integration just as it did to the old scorecard.

## Pending external inputs

Before certain features can be built, these are needed:
- **Sandra's conversation flow** — the WhatsApp chatbot script/prompt
- **Business portfolio field template** — Sandra's definition of what fields a portfolio contains
- **WhatsApp Business API credentials** — phone number verified with Meta or Twilio
- **Serasa API access** — registration required; reference doc is `Serasa Experian developer API's.docx`
