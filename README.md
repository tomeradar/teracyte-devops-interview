# DevOps & MLOps Engineer — Take-Home Assignment

You have **48 hours to submit**. Spend **4–5 hours** on the required tasks, then stop and document anything unfinished. No prior Argo, Kubernetes or AKS experience, cloud account, or paid service is required.

We interview candidates across Junior-Mid to Mid-Senior levels. Start with a clean, working baseline. If a requirement is beyond your experience, implement a simplified version and explain the missing production work.

## The system

Camera images pass through preprocessing, mock model inference and database aggregation. MongoDB tracks experiments; PostgreSQL stores results. FastAPI and Celery serve client requests. Our Core Dev and Data/ML teams release independently. We run on Azure today and need cost-conscious, portable infrastructure.

## Start here

Follow the [local-development guide](docs/local-development.md) to run the supplied Python code and database services. Containerization, E2E verification and CI are your implementation tasks. The sequential analysis runner is acceptable.

## Required tasks

### [Containerize the workloads](tasks/containerize-workloads/README.md)
- Run the API, worker and analysis pipeline with Compose.
- Configure connections externally; keep data and models outside images.

### [Verify an experiment](tasks/verify-experiment/README.md)
- Check five results, experiment status, model digest and execution time.
- Demonstrate one failing check.

### [Support Dev and Data/ML delivery](tasks/support-teams/README.md)
- Add separate workflows for application and analysis/model changes.
- Reuse verification and explain promotion and rollback.

### [Plan a cost-conscious cloud deployment](tasks/cloud-deployment/README.md)
- Map the workloads to cloud infrastructure.
- Compare costs using the supplied worksheet and explain portability.

## Optional extensions

Choose only what interests you after the required work. Completing every extension is not expected.

- [Add Argo orchestration](tasks/argo-orchestration/README.md)
- [Add portable object storage and model caching](tasks/portable-storage-models/README.md)
- [Implement cloud infrastructure as code](tasks/cloud-infrastructure/README.md)
- [Isolate shared development environments](tasks/dev-isolation/README.md)
- [Demonstrate failure recovery](tasks/failure-recovery/README.md)

## AI use and submission

**You are encouraged to use AI assistants. However, unverified, generic AI output will fail the review.**

Submit your repository with:
- **EVIDENCE.md:** exact run commands, relevant outputs, one failed verification and any setup limits.
- **ARCHITECTURE_AND_DECISIONS.md:** a short paragraph per major choice explaining why, one alternative and the trade-off. Include the cloud task's diagram/table and calculations here; no separate architecture essay is needed.

Explain one issue you encountered and how you verified the fix. Label estimates and incomplete work honestly. Keep generated data, model weights and credentials out of Git. We will discuss your working system, how it supports both teams, and the reasoning behind your cloud choices.
