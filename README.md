# DevOps & MLOps Engineer — Take-Home Assignment

You have a **48-hour submission window**. The **MUST-HAVE** work is scoped for about **5–7 hours of focused engineering with AI assistance**. Start with a working baseline, then improve it in small, verifiable steps.

> We interview candidates across Junior-Mid to Mid-Senior levels. Focus on getting a clean, working end-to-end baseline first. If a specific sub-requirement is beyond your current experience, implement a working simplified version and clearly document in **`ARCHITECTURE_AND_DECISIONS.md`** what is missing and how you would build it in production.

Our platform receives camera images, tracks experiments in MongoDB, runs Python analysis steps through Argo Workflows on Kubernetes, and stores tabular results in PostgreSQL. FastAPI and Celery serve client requests. The Core Dev team changes the API, worker and UI; the Data/ML team changes analysis steps, workflow templates and models. We currently run on Azure, but storage and analysis workloads must remain movable to AWS or on-prem GPU Kubernetes when prices change.

## AI policy and decision log

**You are encouraged to use AI assistants. However, unverified, generic AI output will fail the review.** You may use LLMs, Copilot or Cursor. Be ready to run your code and explain the choices and failures.

Submit **ARCHITECTURE_AND_DECISIONS.md**. Keep it focused:

- For each significant choice—storage access, model delivery, DAG shape, database fan-in, CI split, node placement and observability—write **one short paragraph**: why this design, one or two alternatives, and the trade-off. A working simplified implementation plus an honest production follow-up is acceptable.
- Do **two** deeper cost calculations, using **500,000 images/day** and **2,000 workflows/day**: (1) raw-image and mask storage tiering, including Hot/Cool/Archive capacity and retrieval; (2) Spot versus On-Demand analysis compute and the bandwidth/time cost of caching versus repeatedly downloading a **1.5 GiB model**. State image size, retention, workflow shape, cache-hit assumptions, simple formulas, units and a dated price source. Estimates are fine when labeled.
- Add **one cohesive “48-Hour Cloud Exit” section** for the Data/MLOps and storage layers. Show Azure → AWS and Azure → on-prem GPU Kubernetes: what endpoint, URI, registry, secret and infrastructure settings change; how artifacts/data move; rough engineer hours and elapsed time; verification and rollback. Identify any Python or Argo template edits. Aim for configuration-only changes, and call out limitations honestly.
- Include a short evidence table with commands, observed results and links to logs or outputs. Separate measurements from estimates.

Do not commit credentials, generated images or model weights. We value a working, explainable submission over a large set of unrun manifests.

## Starter repository

~~~text
.
├── .gitignore
├── README.md
├── compose.yaml
├── requirements-local.txt
├── pipelines/
│   ├── __init__.py
│   └── steps/
│       ├── __init__.py
│       ├── preprocess/
│       │   ├── __init__.py
│       │   ├── requirements.txt
│       │   └── run.py
│       ├── ml_infer/
│       │   ├── __init__.py
│       │   ├── requirements.txt
│       │   └── run.py
│       └── db_update/
│           ├── __init__.py
│           ├── requirements.txt
│           └── run.py
├── scripts/
│   ├── __init__.py
│   ├── generate_mock_data.py
│   └── run_local.py
├── services/
│   ├── __init__.py
│   └── api/
│       ├── __init__.py
│       ├── main.py
│       ├── requirements.txt
│       └── tasks.py
└── tests/
    ├── test_aggregation.py
    └── test_starter.py
~~~

The starter is a **sequential local baseline**, not the target Argo solution. It generates five 3072 × 2048 images and a deterministic 50 MiB mock weight file. The weight file is bytes, not a valid ONNX model; inference computes deterministic mask metrics and records the model SHA-256. Generated data is ignored by Git.

Requirements: Python 3.11+ and Docker with Compose. The development services bind to loopback on ports 15432 (PostgreSQL), 27018 (MongoDB) and 16379 (Redis). Set POSTGRES_URL, MONGO_URL, CELERY_BROKER_URL or CELERY_RESULT_BACKEND if you need other endpoints. From the repository root:

~~~bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -r requirements-local.txt
docker compose up -d --wait
python -m pytest -q
python -m scripts.run_local --experiment-id demo-local
~~~

The runner should end with five written rows and a COMPLETED experiment. Inspect both databases:

~~~bash
docker compose exec -T postgres psql -U starter -d experiments -c "SELECT experiment_id, count(*) FROM image_metrics GROUP BY experiment_id;"
docker compose exec -T mongo mongosh --quiet experiments --eval 'db.experiments.findOne({_id:"demo-local"})'
~~~

To inspect the serving layer, start the API and Celery worker in separate terminals, then call the endpoints:

~~~bash
python -m uvicorn services.api.main:app --reload
python -m celery -A services.api.tasks.celery_app worker --loglevel=INFO
curl http://127.0.0.1:8000/experiments/demo-local/results
curl -X POST http://127.0.0.1:8000/experiments/demo-local/exports
~~~

The API can register a fresh experiment with POST /experiments and a JSON body containing the five image IDs. It currently checks a fixed local storage path; the Celery export also writes to a fixed path. These are deliberate portability exercises. The development API has no authentication and Compose uses disposable credentials; do not expose them publicly.

### Data contract and starter limitations

Preprocess writes a PNG mask and preprocess.json with experiment_id, image_id, mask_path, threshold, dimensions and foreground count. Inference reads that metadata plus the image and model, then writes inference.json with experiment_id, image_id, foreground_fraction, mean_intensity and model_sha256. The database step expects exactly one inference result per registered image, upserts PostgreSQL rows keyed by (experiment_id, image_id), and marks the MongoDB document COMPLETED. Preserve those identifiers, field meanings and the exact-set completion check when changing transport.

Preprocess deliberately makes full uncompressed pixel copies. **Either** optimize the line **or** measure it and choose defensible Kubernetes memory requests/limits; explain your trade-off. The runner is serial, the model is read on every inference invocation, schema creation happens at runtime, and a PostgreSQL commit followed by a MongoDB failure can leave partial state. These gaps give you concrete design choices; you do not need to solve every production concern in code.

## MUST-HAVE tasks — 100 points

### Part 1 — Containers, portable MLOps and Argo (35 points)

**1.1 Container images (8 points).** Add working Dockerfiles for api, preprocess, ml_infer and db_update. Use a non-root runtime user, dependency-layer caching and a .dockerignore that excludes data, secrets and model weights. Show one build/smoke command for each image; the API image must also run a Celery worker. Multistage builds and further size optimization are welcome when useful, but a clear working image is the priority.

**1.2 Portable storage and model delivery (12 points).** Replace fixed storage paths with **one configured storage approach**:

- A cloud-neutral URI layer such as fsspec, universal_pathlib or smart-open, where local, s3:// and az:// locations are selected through environment configuration; provider packages may live behind that layer, but Azure SDK calls must not enter business logic; **or**
- Standard S3 access such as boto3/s3fs against **MinIO**, deployed as a portable object store on AKS, AWS or another Kubernetes provider.

Use MinIO locally/in-cluster for Argo's S3-compatible artifact repository. **Do not build an Azure Blob S3 gateway.** Do not use azure-storage-blob in business logic, Azure Event Grid hardcoding, CosmosDB APIs, Azure ML, or an Azure-only model registry. For the mock model, implement an immutable version or SHA-256 check and show that a bad digest fails. Explain one portable production delivery/cache approach (for example OCI artifacts, MLflow, or a prewarmed object/PVC cache) for a **1.5 GiB model updated five times per week**. You need not run a 1.5 GiB download or build a distributed cache in this assignment, but production Pods must have a credible way to avoid downloading it on every start.

**1.3 Argo Workflow (15 points).** Add workflow-template.yaml with experiment_id and a JSON batch of image_ids. Fan out across the five images; inference for an image must wait for that image's mask, while other images may run in parallel. Fan in to a bounded database update so PostgreSQL is not hit by one connection per analysis Pod. Configure Argo artifacts against MinIO, use stable object keys or artifacts between Pods, and set reasonable per-step requests/limits, retry policy and a concurrency bound. Run the workflow once against the five mock images and record Argo's status/output. Explain what happens if one image fails. A simple, correct DAG is enough; production-grade caching, scheduling and failure recovery may be described in the decision log.

### Part 2 — Practical dual CI/CD and E2E proof (30 points)

**2.1 Two team workflows (10 points).** Create exactly two GitHub Actions files: **.github/workflows/ci-core-dev.yml** and **.github/workflows/ci-data-ml.yml**. Use path filters and/or workflow_dispatch inputs. Core Dev covers services/api, Celery and a future services/web UI path; do not build a React app solely for this assignment. Data/ML covers pipelines/steps, workflow-template.yaml and a model version/digest input. Each workflow should run relevant tests, container builds and available lint/manifest checks. Demonstrate or document a **model-version bump that does not rebuild the API** and a Core Dev change that does not roll the model. Publishing and deployment may be shown as a safe, credential-free dry run with documented promotion/rollback commands.

**2.2 E2E verification and time gate (20 points).** Add **tests/e2e_verify.py**. It must query real PostgreSQL and MongoDB, assert the exact five expected (experiment_id, image_id) rows, MongoDB status COMPLETED and result_count=5, and the expected pinned model SHA-256. Measure from **workflow submission to terminal completion** (or sequential-runner start to completion for Compose) with a monotonic clock and fail at **180 seconds or more**; cluster setup and image builds are outside this interval. Check Argo's native workflow terminal status and duration for the Kubernetes run. All Argo step Pods must have Kubernetes resources.requests and resources.limits; **no continuous cgroup scraper or peak-RSS gate is required**.

Choose either accepted execution path:

1. **Full CI path:** GitHub Actions starts Kind/k3d, MinIO, MongoDB, PostgreSQL and Argo; submits the real Workflow; runs the verifier and time gate.
2. **Local Argo path:** Provide one reproducible **./scripts/run_k8s_e2e.sh** command that starts the required local cluster/services, submits the real Workflow and runs the verifier. Commit **EVIDENCE.md** with the command, Argo workflow output/status, verifier output and runner details. GitHub Actions must still run container builds, linting and a **Docker Compose E2E** using the sequential runner plus the same database/model/time assertions.

Both paths need a real five-image Argo run and a reproducible Compose baseline. Full Kind + Argo execution inside GitHub Actions earns additional credit; CI runner memory instability should not block an otherwise sound submission. Unit tests alone do not count as E2E.

### Part 3 — Focused IaC, cost and cloud exit (25 points)

**3.1 Focused Azure Terraform (13 points).** Add a small, readable **infra/** example for the parts that show the core operating choices: AKS system pool, an analysis **Spot pool that can scale to zero**, and Blob lifecycle rules that distinguish raw images from masks and cover Hot → Cool → Archive/expiry where appropriate. Connect the Argo analysis Pod's Spot toleration or node selector to that pool. Use variables for environment-specific values and show terraform fmt and validate output; do not apply to a real account. You may treat PostgreSQL, MongoDB, network and secret stores as documented external inputs rather than provisioning every managed service. Explain how MinIO or the URI layer fits alongside Azure Blob and what changes on another cloud.

**3.2 Cost and 48-hour exit (12 points).** Put the **two cost calculations** described above and the **single cloud-exit section** in ARCHITECTURE_AND_DECISIONS.md. Cover Data/MLOps and storage first: model/artifact location, credentials, object data movement, Kubernetes placement and rollback. State the likely elapsed time and the specific config/IaC changes for Azure → AWS and Azure → on-prem GPU. Keep Python business logic and the Argo template provider-neutral; if your chosen URI or artifact strategy needs a manifest or code edit, identify it rather than claiming zero changes.

### Part 4 — Short architecture deep-dives (10 points)

Answer these in ARCHITECTURE_AND_DECISIONS.md. Diagrams or a short table are welcome; implementation is optional.

1. **Ephemeral Pod observability and right-sizing (5 points):** how would you capture CPU, memory, duration, OOM and eviction signals for short-lived Pods, correlate them to an experiment without creating high-cardinality metric labels, and choose requests/limits from measurements? Specifically address Pods that finish between Prometheus scrapes. Continuous high-frequency cgroup sampling and a peak-memory CI gate are **bonus-level**, not required.
2. **Pod cold starts and database connection storm (5 points):** with **300 concurrent analysis Pods**, show simple connection and model-download capacity math. Explain bounded fan-in/PgBouncer or equivalent, and one way to reduce Pod/model cold-start cost.

## BONUS tasks — up to 20 extra points

**A. Shared cloud dev isolation (up to 12).** Show how **10 developers and data scientists** can test changes to individual API or analysis steps against one shared dev AKS cluster without overwriting each other or paying for 10 clusters. A **detailed blueprint** in ARCHITECTURE_AND_DECISIONS.md is acceptable; working namespace, routing, quota and cleanup manifests/configs earn more credit. Explain data and artifact partitioning and estimate idle/peak cost.

**B. Spot eviction and duplicate-delivery resilience (up to 8).** Inject a mid-experiment Spot eviction or process kill and provide the recovery trace. Show that retries avoid duplicate PostgreSQL rows, false MongoDB completion and rerunning an already completed heavy inference step; cover Celery export retry behavior as well.

## Submission and evaluation

Submit the repository, ARCHITECTURE_AND_DECISIONS.md and EVIDENCE.md. Include exact commands, observed output, tool versions and any setup limits. Keep generated data and credentials out of Git. We will review the working baseline first, then the Argo run, CI split, portability, cost reasoning and production follow-ups. A clearly documented simplified implementation receives credit; generic AI prose without runnable proof does not.

Point weights are **Part 1: 35, Part 2: 30, Part 3: 25, Part 4: 10**, plus up to **20 bonus**. The highest-value evidence is five correct rows, a COMPLETED experiment, the expected model digest, a successful Argo workflow, and an enforced time gate. We may ask you to explain a model-version change, one failed image or a cloud move during review.
