# Pinecone Semantic Search as an Airflow DAG

## Overview

This project orchestrates an end-to-end pipeline:

1. **Download** a public text dataset  
2. **Process** it into a paragraph CSV (input to vector DB)  
3. **Create** a Pinecone index  
4. **Embed** with SentenceTransformers  
5. **Upsert** vectors into Pinecone  
6. **Search** the index and log top-K results

## Repository Layout

```
.
├─ dags/
│  └─ build_pinecone_search.py        # DAG for the pipeline
├─ docker-compose.yaml                # Your compose file (edited per this README)
├─ screenshots/                       # Put all HW screenshots here
├─ logs/                              # Airflow logs (gitignore recommended)
├─ plugins/                           # Optional plugins (gitignore recommended)
├─ config/                            # Optional Airflow config
└─ README.md                          # This file
```

---

## 1) Docker Compose Setup

### 1.1 Create a `.env` file (fix UID warning & permissions)

**Linux / macOS**
```bash
echo "AIRFLOW_UID=$(id -u)" > .env
echo "AIRFLOW_GID=0"       >> .env
echo "AIRFLOW_PROJ_DIR=$(pwd)" >> .env
```

**Windows (PowerShell)**
```powershell
"AIRFLOW_UID=50000" | Out-File -Encoding ascii -FilePath .env
"AIRFLOW_GID=0"     | Out-File -Append -Encoding ascii -FilePath .env
"AIRFLOW_PROJ_DIR=$((Get-Location).Path)" | Out-File -Append -Encoding ascii -FilePath .env
```

### 1.2 Ensure volumes & user mapping in `docker-compose.yaml`

- Mount standard Airflow paths:
  ```yaml
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
  ```
- Run containers as your UID (not root):
  ```yaml
  user: "${AIRFLOW_UID}:0"
  ```

### 1.3 Add required Python packages (runtime services only)

In the **shared environment** (often at `x-airflow-common.environment` or similar) set:

```yaml
_PIP_ADDITIONAL_REQUIREMENTS: >-
  sentence-transformers
  pinecone[grpc]
  apache-airflow-providers-pinecone
```

> These install on the **running** Airflow services (webserver/scheduler), not during init.

### 1.4 Keep `airflow-init` light (important)

For the `airflow-init` service **only**, clear the requirements so init doesn’t try to pip heavy packages:

```yaml
airflow-init:
  environment:
    <<: *airflow-common-env
    _AIRFLOW_DB_MIGRATE: 'true'
    _AIRFLOW_WWW_USER_CREATE: 'true'
    _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
    _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
    _PIP_ADDITIONAL_REQUIREMENTS: ""   # keep init fast/reliable
```

> If your compose already contains these keys, just ensure `_PIP_ADDITIONAL_REQUIREMENTS` is `""` under `airflow-init`.

---

## 2) Bring Up Airflow

```bash
docker compose down -v
docker compose pull
docker compose up airflow-init
docker compose up -d
```

Airflow UI: **http://localhost:8081**  
Default login (if created by init): **airflow / airflow**

If the user wasn’t created automatically:
```bash
docker compose exec airflow airflow users create   --username airflow --password airflow   --firstname Admin --lastname User   --role Admin --email admin@example.com
```

---

## 3) Configure Airflow Variables (Pinecone)

Airflow UI → **Admin ▸ Variables** → add:

- `PINECONE_API_KEY` = `YOUR_REAL_KEY_HERE`
- `PINECONE_INDEX_NAME` = `class_demo_idx` (or your preferred name)

> Mask your API key value in screenshots.

---

## 4) Add the DAG

Create the file `dags/build_pinecone_search.py` with this content:

```python
from __future__ import annotations
import os, time, csv, requests
from datetime import datetime
from typing import List, Dict

from airflow import DAG
from airflow.decorators import task
from airflow.models import Variable

from sentence_transformers import SentenceTransformer
from pinecone import Pinecone, ServerlessSpec

DATA_DIR = "/opt/airflow/data"
RAW_TXT = os.path.join(DATA_DIR, "romeo_juliet.txt")
INPUT_CSV = os.path.join(DATA_DIR, "paragraphs.csv")

MODEL_NAME = "sentence-transformers/all-MiniLM-L6-v2"  # 384 dims
INDEX_DIM = 384
INDEX_METRIC = "cosine"
DEFAULT_INDEX = Variable.get("PINECONE_INDEX_NAME", default_var="class_demo_idx")

default_args = {"owner": "airflow", "retries": 1}

with DAG(
    dag_id="build_pinecone_search",
    start_date=datetime(2025, 1, 1),
    schedule_interval=None,
    catchup=False,
    default_args=default_args,
    tags=["pinecone", "vectors", "nlp"],
) as dag:

    @task
    def download_data():
        os.makedirs(DATA_DIR, exist_ok=True)
        url = "https://www.gutenberg.org/cache/epub/1513/pg1513.txt"
        r = requests.get(url, timeout=60)
        r.raise_for_status()
        with open(RAW_TXT, "w", encoding="utf-8") as f:
            f.write(r.text)
        return RAW_TXT

    @task
    def process_data(raw_path: str):
        with open(raw_path, "r", encoding="utf-8", errors="ignore") as f:
            text = f.read()
        paragraphs = [p.strip().replace("
", " ") for p in text.split("

")]
        paragraphs = [p for p in paragraphs if len(p.split()) > 8]

        with open(INPUT_CSV, "w", newline="", encoding="utf-8") as f:
            w = csv.writer(f)
            w.writerow(["id", "text"])
            for i, p in enumerate(paragraphs):
                w.writerow([f"rj-{i:05d}", p])
        return INPUT_CSV

    @task
    def create_index():
        api_key = Variable.get("PINECONE_API_KEY")
        pc = Pinecone(api_key=api_key)
        index_name = DEFAULT_INDEX

        if not pc.has_index(index_name):
            pc.create_index(
                name=index_name,
                dimension=INDEX_DIM,
                metric=INDEX_METRIC,
                spec=ServerlessSpec(cloud="aws", region="us-west-2"),
            )
            while True:
                desc = pc.describe_index(index_name)
                status = getattr(desc, "status", {})
                if status and status.get("ready"):
                    break
                time.sleep(2)
        return index_name

    @task
    def embed_and_upsert(index_name: str, input_csv_path: str):
        api_key = Variable.get("PINECONE_API_KEY")
        pc = Pinecone(api_key=api_key)
        index = pc.Index(index_name)

        rows: List[Dict] = []
        with open(input_csv_path, encoding="utf-8") as f:
            r = csv.DictReader(f)
            for row in r:
                rows.append(row)

        model = SentenceTransformer(MODEL_NAME)

        batch_size = 128
        total = len(rows)
        for start in range(0, total, batch_size):
            chunk = rows[start : start + batch_size]
            texts = [c["text"] for c in chunk]
            vecs = model.encode(texts, show_progress_bar=False, normalize_embeddings=True)

            vectors = []
            for item, vec in zip(chunk, vecs):
                vectors.append({
                    "id": item["id"],
                    "values": vec.tolist(),
                    "metadata": {"source": "rj", "n_tokens": len(item["text"].split())},
                })

            index.upsert(vectors=vectors)

        return f"Upserted {total} paragraphs into {index_name}"

    @task
    def run_search(index_name: str, query: str = "love at night on the balcony"):
        api_key = Variable.get("PINECONE_API_KEY")
        pc = Pinecone(api_key=api_key)
        index = pc.Index(index_name)

        model = SentenceTransformer(MODEL_NAME)
        qvec = model.encode([query], normalize_embeddings=True)[0].tolist()

        res = index.query(vector=qvec, top_k=5, include_metadata=True)
        print("=== QUERY:", query)
        for m in res.matches:
            print(f"score={m.score:.4f} id={m.id} meta={m.metadata}")
        return "done"

    raw = download_data()
    prepared = process_data(raw)
    idx = create_index()
    up = embed_and_upsert(idx, prepared)
    _ = run_search(idx)
```

---

## 5) Run the DAG

1. Open **http://localhost:8081**, turn the DAG **On**  
2. Click **Trigger DAG** (▶)  
3. Wait for all tasks to succeed  
4. Open **Logs** for each task and capture screenshots (see Deliverables)

---

## 6) Deliverables (Screenshots + Files)

Include these in your repo under `screenshots/` and in your report:

1. **docker-compose edits** — showing `_PIP_ADDITIONAL_REQUIREMENTS` with:
   - `sentence-transformers`, `pinecone[grpc]`, `apache-airflow-providers-pinecone`
2. **Airflow up/healthy** — UI reachable at `http://localhost:8081`
3. **Airflow Variables** — `PINECONE_API_KEY` (masked) and `PINECONE_INDEX_NAME`
4. **DAG listed** — `build_pinecone_search` visible
5. **Graph view** — full task chain
6. **Logs: download_data**
7. **Logs: process_data**
8. **Logs: create_index**
9. **Logs: embed_and_upsert**
10. **Logs: run_search** — includes top-K results in the log

Also submit:
- `docker-compose.yaml` (part of your GitHub submission)
- This `README.md`
- The **GitHub repo link** (below)

---

## 7) Quick Verification

```bash
# Confirm packages installed in the running container
docker compose exec airflow python -c "import sentence_transformers, pinecone; print('ok')"
```

If your DAG does not appear, confirm the file exists at `./dags/build_pinecone_search.py`, the mounts are correct, and refresh the UI.

---

## 8) Troubleshooting

- **airflow-init exits 1**  
  Ensure `_PIP_ADDITIONAL_REQUIREMENTS: ""` under **airflow-init**. Heavy package installs should be on runtime services only.

- **ModuleNotFoundError (sentence_transformers / pinecone)**  
  Confirm shared env has the packages and **restart**:
  ```bash
  docker compose down -v
  docker compose up -d
  ```

- **AIRFLOW_UID warning**  
  Ensure `.env` exists (Section 1.1) and your services use `user: "${AIRFLOW_UID}:0"`.

- **Index not ready**  
  The DAG waits in a loop; give new serverless indexes ~10–60s.

---

## 9) GitHub Link

Add your repository URL here once pushed:

**Repo:** `<https://github.com/your-username/your-repo>`

---

## 10) Notes

- Embedding model: **all-MiniLM-L6-v2** (384-dim)  
- Pinecone metric: **cosine**  
- Region in example: **aws / us-west-2** (adjust as needed)  
- Airflow UI mapped to host **8081** (`"8081:8080"` in compose)  

Good luck, and remember to include **all screenshots** with clear task logs for full credit!
