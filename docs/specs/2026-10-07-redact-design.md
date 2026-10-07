# redact: Design Spec

**Date:** 2026-10-07
**Status:** Design agreed in conversation; implementation plan not yet written
**Author:** Nur Ikhmal Hakim (design decisions), drafted with Claude

---

## 1. Summary

`redact` is an open-source tool that takes a PDF and returns a copy with personal data (PII)
**truly removed**, not just covered with black boxes. Detection runs **locally**, so documents never
leave the user's machine (or the user's own AWS account in cloud mode).

It has two parts:

- **Engine (Python):** detects PII with OpenAI's open-weight Privacy Filter model plus
  Malaysia-specific pattern rules, then redacts the PDF with PyMuPDF.
- **CLI (Go):** a single downloadable binary, `redact`, that batch-processes folders. It either
  sends files to a local engine, or uploads them to a cloud pipeline (S3 → SQS → Lambda) and
  downloads the results.

## 2. Goals

1. **Useful to real people:** anyone can run `docker compose up` and `redact ./docs --out ./clean`
   offline, and nothing leaves their machine.
2. **Provable redaction:** every output PDF is re-checked, and the tests prove that the redacted
   text cannot be extracted.
3. **Malaysian coverage:** MyKad numbers, Malaysian passports and phone formats are detected (the
   model has no government-ID category), and an eval measures how the model handles Malaysian data.
4. **Portfolio evidence for cloud/DevOps roles:**
   - Terraform-managed AWS pipeline
   - CI/CD with GitHub Actions (OIDC, no stored keys)
   - SQS with a dead-letter queue (DLQ), retries and a concurrency cap
   - CloudWatch observability
   - multi-platform Go release automation
5. **AI evidence, as an add-on:** local model integration plus a measured eval, not a demo-only
   integration.
6. **Near-zero cost:** deploy, record a demo, `terraform destroy`. Nothing runs between demos.

## 3. Non-goals (version 1)

- Scanned or image-only PDFs (would need OCR). These are detected and rejected with a clear message.
- A web UI or public upload site. This avoids frontend scope, and avoids collecting strangers'
  sensitive documents.
- Non-PDF formats (DOCX, images).
- Always-on hosting, multi-tenant SaaS, user accounts.
- Fine-tuning the model.
- Perfect recall. The tool reports what it found; users remain responsible for reviewing output.

## 4. Users

Built for anyone, as public open source:

| User | How they use it |
|---|---|
| Individuals (e.g. redacting a payslip or IC copy before sharing) | Local mode via CLI + Docker |
| Companies batch-processing documents | Cloud mode, or local mode on a server |
| Developers | Engine HTTP API, or CLI in scripts |

## 5. Key facts about the model

Verified 2026-10-07 from public sources. Items marked **[verify]** must be confirmed during
implementation.

- **Model:** OpenAI Privacy Filter, `openai/privacy-filter` on Hugging Face, released April 2026,
  Apache 2.0.
- **Input and output:** text in; labelled character/token spans out.
- **Categories (8):**
  - `private_person`
  - `private_email`
  - `private_phone`
  - `private_address`
  - `private_url`
  - `private_date`
  - `account_number`
  - `secret`
- **No government-ID category:** MyKad and passport numbers need our own rules.
- **Size:** sparse mixture-of-experts, 1.5B total / ~50M active parameters per token, 128K context.
  Runs on CPU.
- **Language:** English-first. Malay names and addresses are uncharacterised, which the eval covers.
- **[verify]** RAM needed to load the full weights (likely all 1.5B parameters must be resident,
  roughly 3 GB). This determines Lambda memory size and cold-start time.
- **[verify]** Supported runtime (transformers version, ONNX availability) and CPU throughput in
  pages per second.

Sources:
- https://openai.com/index/introducing-openai-privacy-filter/
- https://grepture.com/blog/openai-privacy-filter-pii-redaction

## 6. Architecture

### 6.1 Local mode

```
redact (Go CLI)
  └─ precheck each file (is PDF? text layer? size? hash)
  └─ goroutine worker pool (N workers, retries, progress)
        └─ HTTP POST /redact ──► engine container (Python, docker compose)
                                    ├─ extract text + char positions (PyMuPDF)
                                    ├─ detect: model spans ∪ MY rules spans
                                    ├─ map spans → page rectangles
                                    ├─ apply true redaction + strip metadata
                                    └─ verify: re-extract, assert no PII remains
        ◄── redacted PDF + JSON report
  └─ write ./clean/*.pdf + summary
```

### 6.2 Cloud mode (deployed only for the demo)

```
redact --cloud ./docs
  └─ goroutines: parallel upload ──► S3 input bucket
                                        └─ S3 event ──► SQS queue ──► Lambda (same engine image)
                                                          │               ├─ success ──► S3 output bucket
                                                          │               └─ failure ──► retry ──► DLQ
                                                          └─ reserved concurrency cap (cost control)
  └─ goroutines: poll output bucket, parallel download
CloudWatch: logs (no document content), metrics, alarm on DLQ depth
AWS Budget alert on the account
```

### 6.3 Language split rationale

To be written up as a decision record (`docs/decisions/0001-python-engine-go-cli.md`):

- **Python** for the engine, because the model runtime and the best PDF redaction library
  (PyMuPDF) live there.
- **Go** for the CLI, because a single static binary is easy to distribute, cross-compiles to
  Linux, macOS and Windows, and makes concurrent I/O easy with goroutines.
- **Lambda instead of goroutines in the cloud.** Each Lambda invocation handles one file and AWS
  provides the parallelism. Goroutines are used where a long-running process handles many jobs:
  the CLI.
- **HTTP API between them,** giving a clean, testable boundary.

## 7. Components

### 7.1 Engine (Python), `engine/`

| Unit | Responsibility |
|---|---|
| `app/api.py` | HTTP API: `POST /redact` (PDF in → redacted PDF + report), `GET /health` (model loaded?) |
| `app/detect/model.py` | Load the Privacy Filter once at startup; text → labelled spans |
| `app/detect/rules_my.py` | Regex + validation for MyKad (`YYMMDD-PB-####`, with or without dashes, valid date and place-of-birth code), Malaysian passport, MY phone formats |
| `app/detect/merge.py` | Union of model spans and rule spans; merge overlaps; per-category on/off config |
| `app/pdf.py` | PyMuPDF: extract chars with positions → map spans to rectangles (handles line breaks and multi-line spans) → add redaction annotations → `apply_redactions()` → strip metadata/annotations → save |
| `app/verify.py` | Re-extract text from output; fail if any detected PII string is still present |
| `app/lambda_handler.py` | SQS event → download from S3 → same pipeline → upload result + report |
| `Dockerfile` | One image used both as the local server and as the Lambda container image |

**Report (JSON) per file:** counts per category, pages touched, engine version, model version, and
a list of skipped items with reasons. **Never the PII values themselves.**

### 7.2 CLI (Go), `cli/`

| Unit | Responsibility |
|---|---|
| `cmd/redact/main.go` | Flags: input path(s), `--out`, `--workers`, `--cloud`, `--categories`, `--engine-url` |
| `internal/precheck` | Is it a PDF, is it encrypted, does it have a text layer (reject scanned), size limit, SHA-256 |
| `internal/worker` | Goroutine pool, bounded concurrency, retries with backoff, progress bar, end summary |
| `internal/engine` | HTTP client for the local engine |
| `internal/cloud` | S3 upload, poll for output (timeout), download; uses the AWS SDK for Go v2 |
| `.goreleaser.yaml` | Builds for Linux/macOS/Windows (amd64/arm64) with checksums on tag |

**Exit codes:** 0 when all files succeeded, 1 when some failed (with a summary listing them),
2 for usage errors.

### 7.3 Infrastructure (Terraform), `infra/`

- **Storage:**
  - S3 input and output buckets (encryption on, public access blocked, lifecycle rule
    auto-deletes objects after N days)
  - ECR repository for the engine image
- **Processing:**
  - SQS queue + DLQ (`maxReceiveCount` = 3)
  - Lambda from the ECR image; memory sized from the [verify] RAM measurement; timeout sized from
    the throughput measurement; reserved concurrency cap
- **Permissions:**
  - least-privilege IAM roles
  - GitHub OIDC provider and deploy role (no long-lived AWS keys)
- **Monitoring and cost:**
  - CloudWatch log groups with retention; alarm on DLQ messages > 0
  - AWS Budget alert
- **State:** remote Terraform state in S3, with state locking.

### 7.4 Eval, `eval/`

- **Data:**
  - **Synthetic data only.** Generated fake Malaysian PII (names in Malay, Chinese and Indian
    styles, MyKad numbers, addresses, phones, accounts) rendered into PDFs with known ground truth.
  - **Never real documents or real IC numbers.**
- **Comparisons:**
  1. model only
  2. rules only
  3. model + rules
- **Scoring:** per-category precision, recall and F1. Recall matters most: a miss is a leak.
- **Output:** results published in `eval/results/` and summarised in the README.

## 8. Error handling

| Case | Behaviour |
|---|---|
| Not a PDF / corrupt | CLI precheck rejects; engine returns 400 |
| Encrypted PDF | Rejected with a clear message (v1) |
| No text layer (scanned) | Rejected: "scanned PDF, OCR not supported in v1" |
| Oversized file / too many pages | Rejected at precheck; limit configurable |
| Model not loaded | `/health` reports not ready; CLI waits, then fails clearly |
| Verification finds leftover PII | Output **not written**; file marked failed |
| Engine HTTP error / timeout (local) | Retry with backoff, then mark failed; batch continues |
| Lambda failure (cloud) | SQS retry up to 3, then DLQ; alarm fires; CLI reports a timeout for that file |

**Logging rule:** logs and reports never contain document text or PII values. They contain only
file hashes, counts and categories.

## 9. Security and privacy

- Local-first: in local mode, nothing leaves the machine.
- Cloud mode runs in the user's own AWS account. Buckets are private and encrypted, with
  auto-expiry.
- No document content in logs, metrics or reports.
- Secrets: GitHub OIDC to AWS. Secret scanning and push protection enabled. `.gitignore` covers
  `*.tfstate`, `.env` and sample outputs.
- Test and eval data are synthetic only.
- `SECURITY.md` explains how to report vulnerabilities.
- README disclaimer: automated redaction can miss things; review the output before sharing.

## 10. Testing

- **Engine unit tests:**
  - rule patterns, including valid and invalid MyKad dates and codes
  - span merging
  - span → rectangle mapping, including multi-line spans
- **Leak test (core):** for every test PDF, redact → re-extract all text → assert that no
  ground-truth PII string remains, and that metadata is stripped.
- **API tests:** `/redact` and `/health` against the container.
- **CLI tests:**
  - precheck cases
  - worker pool (concurrency limit respected, retries, partial failure exit code) against a fake
    engine
- **Eval:** runs in CI on a small set; full run documented.
- **Infra:** `terraform fmt`, `validate` and `tflint` in CI.

## 11. CI/CD (`.github/workflows/`)

| Workflow | Trigger | Does |
|---|---|---|
| `engine-ci.yml` | PR / push | lint, tests, leak test, Docker build |
| `cli-ci.yml` | PR / push | `go vet`, tests, race detector |
| `infra-ci.yml` | PR touching `infra/` | fmt, validate, tflint, `plan` |
| `release.yml` | tag `v*` | GoReleaser multi-platform binaries + checksums; engine image pushed |
| `deploy.yml` | manual | OIDC to AWS → push image to ECR → `terraform apply` (demo only) |

## 12. Repository layout

```
redact/
├── engine/          # Python engine (see 7.1)
├── cli/             # Go CLI (see 7.2)
├── infra/           # Terraform (see 7.3)
├── eval/            # synthetic data, scoring, results (see 7.4)
├── docs/
│   ├── architecture.md
│   ├── decisions/   # one short ADR per "why", written by the author
│   ├── runbook.md   # deploy, demo, teardown, DLQ handling
│   ├── postmortems/ # break-it-on-purpose write-ups
│   └── specs/       # this file
├── .github/workflows/
├── docker-compose.yml
├── Makefile
├── LICENSE          # Apache-2.0
├── SECURITY.md
└── README.md
```

## 13. Milestones

1. **Engine core, local only:**
   - text extraction with positions
   - MyKad rules
   - true redaction
   - leak test
   - Proves the hardest part (span → rectangle mapping) first.
2. **Model integration:** Privacy Filter loaded once, merged with rules; measure RAM and speed
   (resolves the [verify] items).
3. **Engine API + Docker + docker compose.**
4. **Go CLI, local mode:** precheck, worker pool, summary, exit codes.
5. **Eval:** synthetic Malaysian dataset, three-way comparison, results in README.
6. **Cloud:**
   - Terraform (S3, SQS + DLQ, Lambda, ECR, IAM, CloudWatch, budget)
   - Lambda handler
   - CLI `--cloud`
7. **CI/CD:** all workflows, OIDC deploy, GoReleaser release.
8. **Demo and docs:**
   - deploy → record the demo → destroy
   - architecture diagram
   - decision records
   - runbook
   - one break-it-on-purpose postmortem (e.g. a poison PDF filling the DLQ)

Each milestone should leave the repo in a working, demonstrable state.

## 14. Working agreement (learning goals)

- The author writes the Terraform, the GitHub Actions workflows, the monitoring setup, and all
  decision records and postmortems personally. Claude may review and explain, but not author these.
- Claude may help more with boilerplate: API scaffolding, README formatting, synthetic data
  generators.
- **Debugging rule:** the author attempts each error alone for 20–30 minutes before asking for help.
- **Done test for each piece:** the author can rebuild it from a blank file using only the docs.

## 15. Open questions

1. Model RAM and CPU throughput (milestone 2), which determine Lambda memory and timeout, and
   whether Lambda is viable or ECS Fargate is needed.
2. Model performance on Malay, Chinese and Indian Malaysian names and addresses (milestone 5).
3. Default redaction style: solid black box only, or black box plus category label?
4. Project name / repo name on GitHub (working name: `redact`).
