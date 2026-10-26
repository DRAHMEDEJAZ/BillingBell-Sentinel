# BillingBell Sentinel™

**Verify AI-generated healthcare billing actions before they reach the payer.**

![tests](https://github.com/DRAHMEDEJAZ/billingbell-sentinel/actions/workflows/tests.yml/badge.svg)

AI tools now suggest diagnosis and procedure codes and draft claims for medical practices. They save time, but they can be confidently wrong, and a wrong claim means a denial, rework, compliance risk and sometimes a surprise bill for the patient. 41% of providers report denial rates of 10% or higher ([Experian Health, 2025](https://www.experian.com/blogs/healthcare/state-of-claims-2025/)), and working one denied claim costs about $57 ([HFMA, 2025](https://www.hfma.org/fast-finance/ai-automation-payer-denials-management/)).

Sentinel is a verification layer between any AI billing tool and the payer. It checks every claim, explains each risk in plain language, maps it to the denial code it would trigger, and keeps a trained person in charge of every decision.

Built by [BillingBell Healthcare Systems](https://billingbell.com) (Woodbridge, Virginia) for the Meta Global AI Developer Hackathon 2026, using **Muse Spark through the Meta Model API**.

```
AI suggests a claim → Sentinel checks it → Risk explained → Biller decides → Submission + audit trail
```

## What it checks

**Layer 1: rule engine** (runs offline, no API key). Each rule targets a common denial and names the Claim Adjustment Reason Code (CARC) it prevents.

| Check | What it catches | CARC |
|---|---|---|
| Required fields, NPI | Missing data; NPI that fails the Luhn check digit | 16 |
| Eligibility | Coverage not active on the date of service | 27 |
| Timely filing | Claim past, or close to, the payer's filing limit | 29 |
| ICD-10 / CPT / HCPCS format | Invalid or duplicate codes, bad units, broken diagnosis pointers | 16 |
| Prior authorization | Payer-required authorization number missing | 197 |
| NCCI PTP edits | Bundled code pairs; knows which edits a modifier can and cannot bypass | 97 |
| Medically Unlikely Edits | Units over the per-line or per-day MUE limit (MAI 1, 2, 3) | 151 |
| Modifiers | Missing modifier 25, conflicting modifiers (RT+LT, 26+TC), unknown modifiers | 4 |
| E/M level by time | Documented time that does not support the billed level (over- or undercoding) | 150 |
| New vs established | New-patient code for a patient seen within 3 years | 16 |
| Global period | E/M during a surgical global period without 24/25/57/79 | 97 |
| Age conflicts | Preventive codes outside the patient's age band | 6 |
| Duplicates, charges | Repeated lines, zero or negative charges | 18, 16 |

**Layer 2: AI documentation check** (Muse Spark, Meta Model API). Reads the clinical note and decides, code by code, whether the documentation supports the claim. It quotes the evidence first, returns structured JSON with severity and confidence, and also points out documented services that were never billed.

## Built for production safety

- **PHI guard:** SSNs, phone numbers, emails, record numbers, full dates and ZIP codes are redacted before a note leaves the practice.
- **Prompt-injection defense:** the note is passed as data, and the model is told to ignore instructions inside it.
- **No invented codes:** AI findings about codes that are not on the claim are discarded and counted.
- **Never blocked by AI:** retries with backoff; if the AI is down, rule checks still run.
- **Tamper-evident audit log:** every check is written to a hash-chained log; `verify-log` detects any edit.
- **Human in the loop:** Sentinel never submits a claim and never makes clinical decisions.

## Quick start

```bash
pip install -r requirements.txt

# Check one claim (rules only)
python -m sentinel check examples/DEMO-1003.json --no-ai

# Add the Muse Spark documentation check
cp .env.example .env        # then add your Meta Model API key
python -m sentinel check examples/DEMO-1001.json examples/DEMO-1001.txt --html report.html

# Check a whole folder and produce one HTML report
python -m sentinel batch examples --html report.html --audit-log audit_log.jsonl

# Measure accuracy and run the tests
python -m sentinel benchmark
pytest
```

Status meanings: **CLEAN** = ready for biller sign-off, **REVIEW** = biller decision needed, **HOLD** = fix before submission. Each report has a 0–100 risk score.

## Testing

- **20 automated tests:** rule engine, NPI validation, PHI redaction, AI layer with a mock client (bad JSON, timeouts, invented codes), audit-log tampering, reports and CMS file loaders.
- **24-case labeled benchmark** (`benchmark/cases`), one case per denial type plus clean controls: currently 24/24 exact match, precision 1.00, recall 1.00. These are synthetic cases we wrote, so they confirm each rule works; they are not a measure of real-world accuracy. The next step is an expert-reviewed set of real de-identified claims.
- GitHub Actions runs the tests and benchmark on every push.

## Knowledge base

Sample NCCI and MUE edits ship in `sentinel/data/`. For real use, download the quarterly CMS NCCI PTP and practitioner MUE files and load them with `load_ncci_csv()` and `load_mue_csv()` in `sentinel/knowledge.py`. Payer rules (timely filing, prior authorization) are configured in `sentinel/data/payers.json`.

## Project structure

```
sentinel/rules.py       Rule engine (13 rule groups, CARC mapping)
sentinel/llm.py         Muse Spark documentation check via Meta Model API
sentinel/phi.py         PHI redaction
sentinel/verifier.py    Combines checks, risk score, status
sentinel/audit.py       Hash-chained audit log
sentinel/report.py      Text and HTML reports
sentinel/knowledge.py   NCCI, MUE, payer and coding reference loaders
sentinel/cli.py         check, batch, benchmark, verify-log
examples/               Synthetic claims and notes (no real patient data)
benchmark/cases/        Labeled test claims
tests/                  Automated tests
```

## Roadmap

- SAM 3.1 to extract fields from scanned superbills and paper forms
- Official CMS NCCI/MUE files loaded automatically each quarter
- Expert-reviewed benchmark of real de-identified claims (accuracy and false-alarm rate)
- Web dashboard for billers, and EHR/clearinghouse integration

## Important

Research prototype, not a deployed product. Use synthetic or de-identified data only, and do not send real patient information to any API without a HIPAA-compliant agreement. Sentinel does not provide medical advice or make clinical decisions. CPT® is a registered trademark of the American Medical Association.

**Contact:** research@billingbell.com · [billingbell.com/sentinel.html](https://billingbell.com/sentinel.html)

© 2026 BillingBell Healthcare Systems LLC
