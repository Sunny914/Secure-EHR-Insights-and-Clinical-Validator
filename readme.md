# Secure EHR Insights and Clinical Validator

Patient-scoped retrieval-augmented generation for clinical encounter records. A question is embedded, matched against that patient’s chart in PostgreSQL, stripped of personal identifiers, and answered under a guardrail that refuses new diagnoses and medication advice.

This is a portfolio demonstration of a secure clinical RAG workflow. It is not a medical device, not a diagnostic system, and not a certified HIPAA implementation. Do not use generated answers for patient care.

## What it does

- Loads a cleaned MIMIC-IV encounter extract into PostgreSQL and stores a 768-dimensional clinical embedding on each row.
- Retrieves the closest encounter rows for a selected patient with pgvector cosine distance.
- Redacts personal identifiers with Microsoft Presidio before any text is sent to the language model.
- Answers from the retrieved chart context through NVIDIA NeMo Guardrails and DeepSeek.
- Refuses requests for new diagnoses, prescriptions, or dose changes.
- Serves the workflow as a FastAPI service and a Streamlit chat UI.

## Architecture

```mermaid
flowchart LR
  subgraph client [Client]
    UI["Streamlit chat :8501"]
  end

  subgraph api [Application]
    API["FastAPI :8000"]
    EMB["BioClinical ModernBERT"]
    PII["Presidio PHI redaction"]
    RAILS["NeMo Guardrails"]
  end

  subgraph data [Data]
    DB[("PostgreSQL + pgvector")]
  end

  LLM["DeepSeek chat API"]

  UI -->|"patient id + messages"| API
  API --> EMB
  EMB -->|"768-d query vector"| DB
  DB -->|"top 5 encounter rows"| API
  API --> PII
  PII --> RAILS
  RAILS --> LLM
  LLM --> UI
```

PostgreSQL runs outside the application process. The API reads `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD` at request time. Embeddings are computed locally. Only the redacted prompt leaves the process, and it goes to the DeepSeek API configured in `src/guardrails/config.yml`.

## How a question is answered

The production path is `POST /api/v1/chat`.

1. The UI loads patients from `GET /api/v1/patients`. That query returns only `subject_id` values that already have a `clinical_embedding`.
2. The user asks a question. Streamlit sends the selected patient id and the full message list.
3. The API embeds the latest question with `NeuML/bioclinical-modernbert-base-embeddings` (768 dimensions).
4. pgvector searches `patient_encounters` with the cosine-distance operator `<=>`, filtered to that `subject_id`, and returns the five nearest rows.
5. Those rows are rendered as medication, dose, route, event, lab test, comments, and diagnosis.
6. Presidio redacts the retrieved text.
7. The redacted chart is prepended to the user question. Earlier chat turns are kept so the guardrail sees the conversation.
8. NeMo Guardrails either refuses medical advice or calls DeepSeek to answer from that context.
9. The UI appends a non-diagnostic disclaimer to the reply.

A second endpoint, `POST /api/v1/clinical-query`, runs the same redaction and guardrail path against a fixed synthetic chart snippet. It returns both the redacted context and the model reply, which makes the privacy step visible without querying the database.

## Safety controls

| Control | Where | Behavior |
| --- | --- | --- |
| Patient scope | `src/api/main.py` | Similarity search is constrained with `WHERE subject_id = :subject_id`. |
| PHI redaction | `src/pii_redaction/presidio_service.py` | Presidio anonymizes `PERSON`, `PHONE_NUMBER`, `EMAIL_ADDRESS`, `US_SSN`, `LOCATION`, `ORGANIZATION`, and `DATE_TIME` before the prompt is built. |
| Custom recognizers | same module | A US SSN pattern (`###-##-####`) and a hospital deny-list (Massachusetts General Hospital, Mayo Clinic, Cleveland Clinic) are registered on the analyzer. |
| Medical-advice rail | `src/guardrails/rails.co` | Requests to prescribe, diagnose, change a dose, or recommend a drug receive a fixed refusal and are not answered as clinical advice. |
| Visible disclaimer | `src/ui/app.py` | Every assistant message is marked as an AI summary that must not be used for diagnosis. |
| Secrets | `.gitignore` | `.env` is ignored. Database credentials and the model API key stay out of the repository. |

Redaction is automated and best-effort. It reduces identifiers in the prompt. It does not make the pipeline a HIPAA-compliant de-identification product. Retrieved clinical facts are still sent to a third-party model after redaction.

The Colang file defines a patient-history intent and a medical-advice intent. The active flow is the refusal flow. History questions fall through to the language model with the retrieved chart as context.

## Data and embeddings

The encounter table is loaded from `data/MIMIC_IV_Trasncript.csv` (232,158 rows). The file is a cleaned MIMIC-IV medical-transcript extract. Source notes are in `data/data.txt`. Use the dataset only under its original terms.

`src/database/schema.sql` creates `patient_encounters` with admission, demographic, medication, order, lab, microbiology, and DRG columns. `scripts/03_apply_vector_schema.py` adds `clinical_embedding vector(768)` after the `vector` extension is installed in PostgreSQL.

Each embedding is one clinical string per encounter row, built from the populated fields below. Comments are capped at 250 characters.

- Admission type
- Prescribed drug
- Lab test name
- DRG severity
- Diagnosis description
- Notes

`scripts/04_generate_embeddings.py` checks that the model output is 768 dimensions, then vectorizes up to 100 rows whose embedding is still null. That bound keeps the demo runnable on a laptop. The retrieval API only searches rows that have an embedding, so the chat patient list follows whatever subset has been vectorized.

## Tech stack

| Layer | Choice |
| --- | --- |
| API | FastAPI, Uvicorn, Pydantic |
| UI | Streamlit |
| Database | PostgreSQL, pgvector, SQLAlchemy, psycopg |
| Embeddings | sentence-transformers, `NeuML/bioclinical-modernbert-base-embeddings` |
| Generation | NeMo Guardrails, DeepSeek (`deepseek-chat`) through an OpenAI-compatible API |
| PHI redaction | presidio-analyzer, presidio-anonymizer, spaCy `en_core_web_lg` |
| Tabular ingest | pandas |

Python 3.12 is the version this project was built with.

## Repository layout

```text
data/
  MIMIC_IV_Trasncript.csv     Encounter extract consumed by ingestion
  data.txt                    Dataset and compliance reference links
scripts/
  01_ingest_baseline_data.py  Apply schema.sql and load the CSV
  02_verify_ingestion.py      Row count and a five-row clinical sample
  03_apply_vector_schema.py   Add clinical_embedding vector(768)
  04_generate_embeddings.py   Encode encounter text and store vectors
  05_test_vector_search.py    Cosine search for a liver / diuretic query
  06_test_guardrails.py       Allowed history question vs refused dose change
src/
  api/main.py                 FastAPI app and the three HTTP endpoints
  database/schema.sql         Relational encounter schema
  guardrails/config.yml       DeepSeek chat model and local embedding model
  guardrails/rails.co         Medical-advice refusal flow
  pii_redaction/presidio_service.py
  ui/app.py                   Streamlit patient picker and chat
requirements.txt
```

Run every command from the repository root. The API imports the `src` package and loads guardrails from `./src/guardrails`.

## Setup

### 1. PostgreSQL and pgvector

Create a database and enable the extension with a superuser:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The ingestion script creates the table. The vector column is added later by `scripts/03_apply_vector_schema.py`. The extension itself must already exist. On Ubuntu this is the `postgresql-14-pgvector` package for PostgreSQL 14.

### 2. Environment

Create a `.env` file in the repository root. Do not commit it.

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ehr_db
DB_USER=your_user
DB_PASSWORD=your_password

OPENAI_API_KEY=your_deepseek_api_key
OPENAI_BASE_URL=https://api.deepseek.com/v1
OPENAI_API_BASE=https://api.deepseek.com/v1
```

NeMo Guardrails uses the OpenAI engine with `base_url` set to DeepSeek in `src/guardrails/config.yml`. The key belongs in `OPENAI_API_KEY`.

The API builds `postgresql+psycopg://...` from those `DB_*` variables. If `DB_HOST` is unset, it falls back to `POSTGRES_URL`, then to `postgresql+psycopg://postgres:password@localhost:5432/clinical_db`.

The numbered scripts build a `postgresql://...` URL from the same variables. That SQLAlchemy URL uses the psycopg2 driver. If a script fails with `No module named 'psycopg2'`, install `psycopg2-binary` in the same environment. The API path already uses the psycopg v3 driver pinned in `requirements.txt`.

### 3. Python environment

```powershell
uv venv --python 3.12 .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

The first API startup downloads BioClinical ModernBERT and loads spaCy and NeMo. Give that process several minutes and enough disk for the model weights.

### 4. Load data and build vectors

```powershell
python scripts/01_ingest_baseline_data.py
python scripts/02_verify_ingestion.py
python scripts/03_apply_vector_schema.py
python scripts/04_generate_embeddings.py
python scripts/05_test_vector_search.py
python scripts/06_test_guardrails.py
python src/pii_redaction/presidio_service.py
```

`01` reads `data/MIMIC_IV_Trasncript.csv`, applies `schema.sql`, and appends rows. Re-running it appends again.

`05` embeds the query “Patient presenting with severe liver disease and fluid retention needing diuretics” and prints the three nearest diagnoses and drugs.

`06` sends a Furosemide history question, which should be answered, and a request to prescribe a higher dose, which should be refused.

The Presidio module prints a synthetic note (name, SSN, hospital, phone, email) and the redacted version.

## Run the application

Start the API first, from the repository root:

```powershell
uvicorn src.api.main:app --reload
```

Interactive docs: [http://localhost:8000/docs](http://localhost:8000/docs)

Then start the UI:

```powershell
streamlit run src/ui/app.py
```

Open [http://localhost:8501](http://localhost:8501). Choose a patient in the sidebar. The select box is searchable, and the sidebar shows which `subject_id` the next retrieval is locked to. Patient ids are cached for five minutes.

## API

### `GET /api/v1/patients`

Returns `{ "patients": ["10000032", ...] }` for distinct subject ids that have embeddings.

### `POST /api/v1/chat`

Production RAG path used by the UI.

```json
{
  "patient_id": "10000032",
  "messages": [
    { "role": "user", "content": "What medications were prescribed to this patient?" }
  ]
}
```

Response:

```json
{
  "status": "success",
  "llm_response": "..."
}
```

`patient_id` is cast to an integer. An unknown patient, or a patient with no embeddings, yields a context string that no historical records were found, and that string is what the model sees.

### `POST /api/v1/clinical-query`

Synthetic-chart path. Useful for checking redaction and the guardrail without similarity search.

```json
{
  "patient_id": "10000032",
  "prompt": "What was the last recorded Furosemide dosage?"
}
```

Response fields: `status`, `redacted_context_used`, `llm_response`.

## Questions to try

These stay inside chart retrieval:

- What medications were prescribed to this patient upon discharge?
- What was the patient’s last recorded dosage?
- What liver-related diagnoses are noted in the patient’s file?
- Was the patient admitted urgently or routinely?
- Show me the historical lab results.

These match the refusal rail:

- Can you prescribe me a new medication?
- Should I increase the patient’s dosage?
- Based on the culture result, what antibiotic should I start?
- Diagnose these symptoms from the chart.

## Deployment shape

The repository ships the application as two processes: FastAPI on port 8000 and Streamlit on port 8501. Streamlit calls the API at `http://localhost:8000`. A single container can start both, publish only 8501, and keep the API on the container network. Point `DB_HOST` at PostgreSQL. Do not publish port 5432 with the application.

No `Dockerfile` is committed in this tree. Add one only at deploy time, pass configuration with `--env-file .env`, and keep `.env` out of the image context.

## Scope of this repository

- Vector search is exact cosine distance over the embedded subset. The schema script does not create an ANN index. That is appropriate for the 100-row embedding demo and is the first thing to add before searching the full 232,158-row extract.
- Embedding text is one composed string per encounter row. Comments longer than 250 characters are truncated before encoding.
- `POST /api/v1/chat` returns the model answer. The retrieved rows stay on the server. `POST /api/v1/clinical-query` is the endpoint that also returns the redacted context it used.
- The API does not authenticate callers. Patient scoping is a query filter, not an authorization layer.
- Startup uses the FastAPI startup event to load Presidio, NeMo, and the embedding model once per process.

## License and use

Follow the license of the MIMIC-IV cleaned transcript dataset for `data/MIMIC_IV_Trasncript.csv`. Generated output is a study aid for this codebase. It is not clinical advice.
