# CoEvolution Knowledge Base

> Storage layer of the CoEvolution Hub (T6.1, AEGIS). The content of every model artifact, dataset and report is stored in **MinIO**, and the metadata describing each file is stored in **MongoDB**.

[![MinIO](https://img.shields.io/badge/MinIO-object%20storage-red)](https://min.io)
[![MongoDB](https://img.shields.io/badge/MongoDB-metadata-green)](https://www.mongodb.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-deployment-blue)](https://kubernetes.io)

---

## Table of Contents

- [Overview](#overview)
- [Why this repository contains no source code](#why-this-repository-contains-no-source-code)
- [Architecture](#architecture)
- [Role in the Testbed](#role-in-the-testbed)
- [MinIO Pools](#minio-pools)
- [Buckets](#buckets)
- [MongoDB](#mongodb)
- [Repository Structure](#repository-structure)
- [Deployment](#deployment)
- [Access](#access)
- [Related Repositories](#related-repositories)

---

## Overview

The Knowledge Base is the central store of the CoEvolution framework. It keeps the evidence produced throughout a model's lifecycle: requirements, assessment and defence results, risk reports, AIMBOMs, trusted models and trusted datasets.

It separates two concerns:

| Store | Holds | Answers |
|-------|-------|---------|
| **MinIO** | The file content, organised in five pools of buckets | *Give me the content of this file* |
| **MongoDB** | A metadata record for every file (pool, bucket, filename, size, content type, uploader, tags), plus the model, version and tool records | *What is this file, where is it, and which files belong to this model version?* |

The two stores are queried differently. A component that already knows a file's identifier only needs its metadata record to locate it. A component looking for "every risk report of this model" searches MongoDB, because structured queries are not practical on raw object storage.

## Why this repository contains no source code

MinIO and MongoDB are mature, widely adopted open-source products and are used **without modification**. All CoEvolution-specific logic is implemented in the **[Orchestrator](https://github.com/CoEvolution-Project-EU/Orchestrator_T6.1)**:

- where each file is stored (which pool and bucket),
- the metadata schemas for models, versions, files and tools,
- access control, including the restriction of each pilot to its own bucket,
- download links, which the Orchestrator generates on demand.

What is specific to CoEvolution at this layer is **how the services are deployed and configured**. This repository therefore contains the **Kubernetes manifests** that define the Knowledge Base on the CoEvolution testbed: images, persistent storage, services, health checks and resource settings. Because they are version-controlled, the Knowledge Base can be redeployed in the same way on any Kubernetes cluster.

## Architecture

```
     Dashboard            Engines (RAGRE, STA, STRAE, SRME)
         │                         │
         └────────── REST ─────────┘
                       │
              ┌────────▼────────┐
              │  Orchestrator   │   ← single entry point to the Knowledge Base
              └───┬─────────┬───┘
       MinIO SDK  │         │  Mongoose
┌─────────────────┼─────────┼─────────────────────────────┐
│ Knowledge Base  ▼         ▼                             │
│  ┌─────┐ ┌────┐ ┌─────┐ ┌────┐ ┌───────┐    ┌─────────┐ │
│  │ SAP │ │ RP │ │ TRP │ │ AP │ │ STRDP │    │ MongoDB │ │
│  └─────┘ └────┘ └─────┘ └────┘ └───────┘    └─────────┘ │
│         MinIO pools (file content)          (metadata)  │
└─────────────────────────────────────────────────────────┘
```

**Single entry point.** No component connects to MinIO or MongoDB directly. Every upload, search and download goes through the Orchestrator's REST API. Access control and metadata consistency are therefore enforced in one place, and the storage layout can change without affecting any client.

**Five independent pools.** Each pool is a separate MinIO deployment with its own storage and credentials, owned by a different CoEvolution partner. This reflects the distributed ownership of models, reports and tools in the project, and allows each pool to be managed independently.

## Role in the Testbed

The Knowledge Base runs on the central CoEvolution server, together with the Dashboard, the Orchestrator, User and Credential Management, and the Communication Infrastructure. It supports every core flow of the platform:

| Flow | What is stored |
|------|----------------|
| Model Registration | The model's requirements and descriptive reports (model information, EU AI Act, datasets) |
| Assessment and Report Generation | Results submitted by users after running assessment and defence tools, plus the reports produced by the engines |
| Risk Assessment | The risk reports produced by RAGRE |
| AIMBOM Generation | The AIMBOM produced by STRAE |
| Trusted Model Registration | Trusted models, together with trusted and pilot data |

## MinIO Pools

| Pool | Full name | Owner | Content |
|------|-----------|-------|---------|
| **SAP** | Security Assurance Pool | CBRL | AIMBOMs |
| **RP** | Report Pool | AEGIS | Reports describing, assessing and defending a model |
| **TRP** | Trust Resources Pool | AEGIS | Model requirements, trusted models, trusted data and pilot data |
| **AP** | Assessment Pool | SLC | Assessment tools and libraries |
| **STRDP** | Security Trust Robustness Defence Pool | UNICA | Defence tools and libraries |

## Buckets

| Pool | Bucket | Purpose |
|------|--------|---------|
| SAP | `ai-mbom` | AIMBOM generated by STRAE |
| RP | `aimodelinfo-reports` | Model name, description, architecture, licence, inputs and outputs |
| RP | `euaiact-reports` | EU AI Act compliance properties |
| RP | `datasets-reports` | Datasets used across design, testing, validation and hardening |
| RP | `risk-reports` | Risk assessment produced by RAGRE |
| RP | `user-assessments` | Results submitted by a user after running an assessment tool |
| RP | `user-defences` | Results submitted by a user after running a defence tool |
| RP | `assessment-reports` | Assessment reports produced by the Security Testing/Assessment Engine |
| RP | `defence-reports` | Defence reports produced by the Security Testing/Assessment Engine |
| RP | `gadgets-reports` | Security and robustness enhancements applied at deployment |
| TRP | `model-requirements` | The model's dependency list, submitted at registration |
| TRP | `trusted-models` | Model artifacts accepted once their evidence chain is complete |
| TRP | `trusted-data` | Curated and generated datasets admitted for reuse |
| TRP | `siemens-data`, `avl-data`, `avi-data` | Data contributed by each pilot; each pilot can access only its own bucket |
| AP | `assessment-tools`, `assessment-libraries` | Assessment tools and libraries |
| STRDP | `design-train-defence-tools`, `deployment-test-defence-tools`, `defence-libraries` | Defence tools and libraries for each lifecycle phase |

## MongoDB

MongoDB holds the platform's structured data in a single application database:

| Collection | Content |
|------------|---------|
| **Models** | Registered models and their versions. Each version keeps the identifiers of the files that belong to it, grouped by report type. |
| **Files** | One metadata record per stored file: pool, bucket, object key, filename, size, content type, uploader and tags. |
| **Tools** | The registry of assessment tools, defence tools and data generators, each linked to its source repository. |

The collections and their indexes are created and managed by the Orchestrator. No data is pre-loaded by this repository.

## Repository Structure

```
├── mongo-db/
│   ├── mongodb-deployment.yaml   # Persistent volume and MongoDB deployment
│   └── mongodb-service.yaml      # MongoDB service
└── pools/
    ├── minio-sap.yaml            # Security Assurance Pool
    ├── minio-rp.yaml             # Report Pool
    ├── minio-trp.yaml            # Trust Resources Pool
    ├── minio-ap.yaml             # Assessment Pool
    └── minio-strdp.yaml          # Security Trust Robustness Defence Pool
```

Each pool manifest defines three resources:

- **PersistentVolumeClaim:** the pool's storage, which persists across restarts.
- **Deployment:** the MinIO server, with readiness and liveness health checks.
- **Service:** exposes the pool's S3 API and web console.

## Deployment

**Prerequisites:** a Kubernetes cluster, `kubectl`, a default StorageClass, and an ingress controller for external access.

**1. Create the credentials.** Credentials are supplied through Kubernetes Secrets, one for MongoDB and one for each MinIO pool, and are **not stored in this repository**:

```bash
kubectl create secret generic minio-<pool>-secret \
  --from-literal=MINIO_ROOT_USER=<user> \
  --from-literal=MINIO_ROOT_PASSWORD=<password>
```

**2. Deploy MongoDB and the MinIO pools:**

```bash
kubectl apply -f mongo-db/
kubectl apply -f pools/
```

**3. Verify:**

```bash
kubectl get pods
kubectl get pvc
```

All pods should be `Running` and ready, and every volume `Bound`.

**Design choices in the manifests:**

- **Persistent volumes:** stored data survives pod restarts and redeployments.
- **Health probes:** Kubernetes routes traffic to a service only when it is ready, and restarts it automatically if it stops responding.
- **Recreate strategy for MinIO:** the old pod is stopped before the new one starts, so two servers never write to the same volume.
- **Resource limits for MongoDB:** the database cannot starve the other Hub components running on the same server.

## Access

| Service | Access |
|---------|--------|
| Orchestrator REST API | `https://aegis.coevolution-project.eu/api/docs`, the interface for all components |
| MinIO consoles | `https://aegis.coevolution-project.eu/{sap,rp,trp,ap,strdp}`, for administrators |

All external traffic goes through the platform's API gateway over HTTPS.

## Related Repositories

| Repository | Description |
|------------|-------------|
| [Orchestrator](https://github.com/CoEvolution-Project-EU/Orchestrator_T6.1) | Backend of the Hub and the only client of the Knowledge Base |
| Communication Infrastructure | Kafka event backbone between the Orchestrator and the engines |

---

## License

CoEvolution Project EU

## Author

**Konstantinos Krouti**
GitHub: (https://github.com/kostaskr13)
