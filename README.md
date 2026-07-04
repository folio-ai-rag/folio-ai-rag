# Folio — Quickstart README

This repository's canonical design lives in Folio_SDD.md. Use that as the source of truth for architecture, data model, and non-functional goals.

This README contains a developer quickstart for running the app locally and the minimal setup required for Phase 1 development.

Quick links:
- Software Design Document: ./Folio_SDD.md
- Implementation plan (feature-by-feature checklist): ./docs/IMPLEMENTATION_PLAN.md

Prerequisites
-------------
- Git
- Docker & Docker Compose (recommended for local infra)
- Python 3.12
- Node 18+ (for the web client)
- An AWS account only if you want to test real S3 or other managed services (not required for local dev)

Repository layout (short)
-------------------------
- apps/api — FastAPI backend (Python)
- apps/web — Next.js frontend (TypeScript)
- packages/shared-types — generated API client types
- docs/IMPLEMENTATION_PLAN.md — prioritized implementation checklist (start here)
- Folio_SDD.md — full design document (source of truth)

Developer quickstart (local dev)
--------------------------------
The goal is to get a working end-to-end vertical slice running locally: Postgres + pgvector, Redis, (optional) MinIO/localstack for S3 emulation, the API, and a worker process.

1) Clone the repo

   git clone https://github.com/folio-ai-rag/folio-ai-rag.git
   cd folio-ai-rag

2) Start local infra with Docker Compose (recommended)

Create a file `docker-compose.dev.yml` with the following minimal services (example):

```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: test
      POSTGRES_DB: folio_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  minio:
    image: minio/minio:latest
    command: server /data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata: