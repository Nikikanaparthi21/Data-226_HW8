# DATA 226 – Week 11 Homework
**Title:** Running Pinecone as an Airflow Job  

---

## Summary
This assignment implements a semantic search pipeline that ingests a text corpus into **Pinecone** using an **Apache Airflow** DAG and validates the workflow by executing a vector search. The pipeline is containerized with Docker Compose and includes required packages (`sentence-transformers`, `pinecone[grpc]`, optional Airflow Pinecone provider).

**What I built**
- A working Airflow DAG (`build_pinecone_search.py`) that:
  - Downloads a text corpus
  - Processes text into paragraph records (CSV)
  - Creates a Pinecone **serverless** index (cosine, 384-dim)
  - Generates embeddings with `all-MiniLM-L6-v2` (SentenceTransformers)
  - Upserts vectors to Pinecone
  - Executes a top‑K semantic search and logs results
- A Compose setup that installs required Python packages in the runtime services.
- Screenshots demonstrating each step and task logs for grading.

---

## Deliverables (Checklist)
- [x] `docker-compose.yaml` includes **sentence-transformers** and **pinecone** packages (1 pt)
- [x] Containers restarted after edits (compose down/up) (screenshots)  
- [x] Airflow **Variable** created for **PINECONE_API_KEY** (and **PINECONE_INDEX_NAME**) (1 pt)
- [x] **Input file** generated for Pinecone (CSV of paragraphs) (2 pts)
- [x] **Pinecone index** created (1 pt)
- [x] **Embeddings + Ingestion** into Pinecone (2 pts)
- [x] **Search** executed against Pinecone (1 pt)
- [x] **Screenshots** captured for each step, including Airflow task logs
- [x] **GitHub repo link** added to this README

---

## Repository Contents
```
.
├─ dags/
│  └─ build_pinecone_search.py        # Airflow DAG for the pipeline
├─ docker-compose.yaml                # Compose with runtime requirements
├─ screenshots/                       # All required screenshots (see list below)
└─ README.md                          # This file
```

---

## Configuration Summary
**Airflow Variables**
- `PINECONE_API_KEY` = ******** (secret)
- `PINECONE_INDEX_NAME` = `class_demo_idx` 

**Vector Config**
- Model: `sentence-transformers/all-MiniLM-L6-v2` → **384** dims
- Index: metric **cosine**, serverless region **aws/us‑west‑2** (adjustable)

**Compose Notes**
- Runtime services install: `sentence-transformers`, `pinecone[grpc]`, `apache-airflow-providers-pinecone`
- `airflow-init` kept light (no heavy pip installs) for reliable DB/user setup
- Airflow UI exposed at **http://localhost:8081**

---

## DAG Overview
**DAG ID:** `build_pinecone_search`

**Tasks:**
1. `download_data` → Fetch corpus and store under `/opt/airflow/data`
2. `process_data` → Split into paragraph rows and write **CSV** (input for embeddings)
3. `create_index` → Create/verify Pinecone index with specified metric/dimensions
4. `embed_and_upsert` → Encode paragraphs → upsert to Pinecone in batches
5. `run_search` → Run a semantic query; log **top‑K** matches

**Artifacts Produced:**
- `/opt/airflow/data/romeo_juliet.txt`
- `/opt/airflow/data/paragraphs.csv` (input to Pinecone)

---

## Results Summary
- **Index name:** `class_demo_idx` (or value from `PINECONE_INDEX_NAME`)
- **Dimension:** 384 • **Metric:** cosine • **Region:** aws/us‑west‑2
- **Vectors upserted:** `<N>`
- **Query used:** `"love at night on the balcony"` (modifiable)
- **Top‑K (sample)**  
  - id=`<rj-00042>` • score=`<0.81>` • meta=`{source:"rj", n_tokens:<…>}`  
  - id=`<rj-00107>` • score=`<0.79>` • meta=`{…}`  
  - id=`<rj-00088>` • score=`<0.78>` • meta=`{…}`  
*(Exact scores/ids appear in the task logs and may vary.)*

---

## Screenshots (Included in `/screenshots`)
1. `01-docker-compose-requirements.png` — Compose shows `sentence-transformers` + `pinecone[grpc]`
2. `02-containers-healthy.png` — Airflow up (or `docker compose ps`)
3. `03-airflow-variables.png` — Variables page (key names visible; API key masked)
4. `04-dag-visible.png` — DAG listed/enabled
5. `05-graph-view.png` — Graph view across all tasks
6. `06-logs-download.png` — Logs for `download_data`
7. `07-logs-process.png` — Logs for `process_data`
8. `08-logs-create-index.png` — Logs for `create_index`
9. `09-logs-embed-upsert.png` — Logs for `embed_and_upsert`
10. `10-logs-run-search.png` — Logs for `run_search` (top‑K results shown)

---

## Reproducibility Notes (Concise)
- Image base: `apache/airflow:2.10.1` (LocalExecutor)
- Required packages installed via runtime `_PIP_ADDITIONAL_REQUIREMENTS`
- Airflow UI at **http://localhost:8081**
- Pinecone key provided via Airflow Variable (`PINECONE_API_KEY`)

---

## Acknowledgments
- Course materials (Week 11)
- Pinecone SDK & SentenceTransformers libraries
- Airflow Docker stack


