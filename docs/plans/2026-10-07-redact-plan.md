# redact: Implementation Plan (rough)

**Spec:** `docs/specs/2026-10-07-redact-design.md`
**Style:** Rough roadmap for a learning project. Each milestone lists what to build, what "done" looks
like, and what to learn. Detail gets worked out while building.

**Who writes what (spec §14):** you write the Terraform, the GitHub Actions workflows, the monitoring,
the ADRs and the postmortems. Claude can help with boilerplate (API scaffolding, data generators,
README formatting) and can review and explain.

---

## Milestone 0: Repo setup (½ day)

- `git init`, `.gitignore` (`*.tfstate`, `.env`, `.venv/`, sample outputs), `LICENSE` (Apache-2.0),
  stub `README.md` and `SECURITY.md`.
- `engine/` as a Python 3.12 project managed with `uv` (deps: `pymupdf`; dev: `pytest`, `pypdf`, `ruff`).
- `Makefile` with `test`, `lint` and `fmt` targets.

**Done when:** `make test` runs an empty test suite.

## Milestone 1: Engine core, local only (the hard part)

Build in this order, test-first:

1. **`app/detect/rules_my.py`**: MyKad (`YYMMDD-PB-####`, with or without dashes, valid date, valid
   place-of-birth code), passport (`A/H/K` + 8 digits) and MY phone regexes. Tests cover valid and
   invalid dates and codes.
2. **`app/detect/merge.py`**: a `Span(start, end, category, source)` type, merging of overlapping
   spans, and per-category on/off. Also consider *propagation*: if a value is detected once, redact
   every occurrence of it.
3. **`app/pdf.py`, extraction**: build one string of the document text from
   `page.get_text("rawdict")`, plus a parallel list that maps each character to its (page, bbox).
4. **`app/pdf.py`, span → rectangles**: group a span's characters by line, giving one rectangle
   per line. That handles multi-line spans.
5. **`app/pdf.py`, redact and strip**: `add_redact_annot` → `apply_redactions()` → delete
   annotations, form widgets and embedded files → `doc.scrub()` → save with `garbage=4, deflate=True`.
6. **`app/verify.py`**: re-extract the output and fail if any detected value (or any rule match)
   is still there. When this fails, the output is **not written**.
7. **`app/pipeline.py`**: `redact_pdf(bytes, config) -> (bytes, report)`, with errors for
   corrupt, encrypted, scanned and over-limit input. Add a `python -m app in.pdf out.pdf` command
   to try it by hand.
8. **Leak test**: generate synthetic PDFs in the tests, redact them, then re-extract with **both**
   PyMuPDF and pypdf and assert that no PII is left, metadata is gone, and nearby non-PII text survives.

**Gotchas found while scoping (worth knowing before you start):**
- Redacting the full character bbox also wipes the lines **above and below**, because glyph boxes
  overlap. A thin band around the x-height (about baseline − 0.45·size to baseline − 0.25·size)
  removes only the target line. Draw the visible black box separately.
- `doc.scrub()` does **not** remove text annotations, and form-field values show up in
  `get_text()`. Delete annotations and widgets explicitly, before extracting.
- An encrypted PDF that needs a password opens with `metadata == None`. Check `needs_pass` first.
- Corrupt input raises `pymupdf.FileDataError`.
- Rotated pages worked in a quick test, but put one in the leak test anyway.

**Done when:** the leak test passes on a set of synthetic PDFs (multi-line values, rotated page,
tight line spacing, metadata, annotations, form fields).

## Milestone 2: Model integration

- `app/detect/model.py`: load `openai/privacy-filter` once and turn text into `Span`s (same
  `Detector` interface as the rules).
- Chunk long documents if needed. Merge model spans with rule spans.
- **Measure and write down:** RAM after load, cold-start time, pages per second on CPU. This
  answers spec §15 Q1 (Lambda vs Fargate).

**Done when:** names, emails and addresses are redacted in the leak test, and the numbers are in
`docs/`.

## Milestone 3: Engine API + Docker

- `app/api.py` (FastAPI): `POST /redact` returns the PDF plus a JSON report (multipart, or the
  report in a header). `GET /health` returns 503 until the model is loaded.
- `Dockerfile` (one image for local use and Lambda), `docker-compose.yml`.
- API tests: 400 for a bad PDF, 422 when verification fails, health readiness.

**Done when:** `docker compose up` works, and `curl` returns a redacted PDF.

## Milestone 4: Go CLI, local mode

- `cli/` Go module. `internal/precheck`, `internal/engine` (HTTP client), `internal/worker`
  (bounded goroutine pool, retries with backoff, progress, summary), `cmd/redact/main.go`.
- Exit codes: 0 when everything succeeded, 1 for a partial failure, 2 for a usage error.
- Tests against an `httptest` fake engine: concurrency cap respected, retries, partial-failure
  exit code. Run with `-race`.

**Done when:** `redact ./docs --out ./clean` processes a folder end to end.

## Milestone 5: Eval

- Generator for synthetic Malaysian PII (Malay, Chinese and Indian names; MyKad numbers;
  addresses; phones; accounts) rendered into PDFs with ground truth.
- Score three setups (model only / rules only / model + rules) with per-category precision, recall
  and F1.
- Results go in `eval/results/` and in a README table.

**Done when:** the results table is in the README, and a small eval runs in CI.

## Milestone 6: Cloud (you write the Terraform)

- `infra/`: remote state, S3 in/out buckets (encryption, public access blocked, lifecycle rule),
  ECR, SQS + DLQ (`maxReceiveCount` = 3), Lambda from the image with reserved concurrency, IAM,
  CloudWatch logs, retention and DLQ alarm, Budget.
- `app/lambda_handler.py`: SQS event → S3 get → `redact_pdf` → S3 put.
- CLI `--cloud`: parallel upload, poll the output bucket with a timeout, download.

**Done when:** `redact --cloud ./docs` works against a deployed stack, then `terraform destroy`.

## Milestone 7: CI/CD (you write the workflows)

- `engine-ci`, `cli-ci`, `infra-ci`, `release` (GoReleaser + engine image), `deploy` (manual, OIDC).

**Done when:** a `v0.1.0` tag produces binaries for every platform, and a deploy runs with no
stored AWS keys.

## Milestone 8: Demo and docs

- Deploy → record the demo → destroy.
- `docs/architecture.md`, ADR 0001 (Python engine / Go CLI), runbook.
- One break-it-on-purpose postmortem (a poison PDF filling the DLQ).

---

## Open decisions to make along the way

- Redaction style: black box only, or black box plus category label (spec §15 Q3). Decide in M1.
- Lambda vs Fargate: decided by the numbers from M2.
- Repo name.
