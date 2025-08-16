---
title: "Face Matching Engine Service"
excerpt: "AI-powered tool to match faces and retrieve similar images from the internet. <br/><img src='/images/proj-2.png'>" 
collection: portfolio
---

## Introduction 
A production‑ready system that finds visually similar faces on the web from a single uploaded photo, then ranks results using learned embeddings and cosine similarity. Built for speed, resilience, and scale with FastAPI and a robust fallback pipeline.

## Overview
* Goal: Given a user‑uploaded image, retrieve visually similar faces from the web and return ranked matches with metadata.
* Core Idea: Combine person detection (YOLOv8) with face detection + embedding (InsightFace) and vector similarity to produce accurate, explainable matches.
* Status: Deployed on AWS with background processing, concurrency controls, and comprehensive error handling.

## Key Features
* One‑click query: Users upload a single image; the system automatically fetches visually similar images from the web.
* Hierarchical detection: YOLOv8 detects people; InsightFace detects faces within detected persons and generates embeddings.
* Semantic ranking: Cosine similarity ranks candidate faces against the query.
* Robust fallbacks: If a detector fails, the pipeline gracefully returns the full image region or switches to a backup model.
* Scalable API: FastAPI backend exposes clean endpoints with background jobs (fire‑and‑forget) for heavy tasks.
* Concurrency safety: Threading with semaphores limits parallelism and protects resources.
* Fault‑tolerant downloads: Resilient web fetchers with retries, timeouts, and circuit‑breaker logic.
* Cloud‑native: Containerized and deployed on AWS with autoscaling in mind.

## Architecture at a Glance
1. Ingress: POST /match (FastAPI) accepts an image upload.
2. Acquisition: A fetcher service queries the web (multiple engines/providers) and downloads candidate images.
3. Storage Layout: Each request is assigned a UUID with separate directories for the query and candidates.
4. Per‑image Pipeline:
    * YOLOv8 Person Detection → person crops.
    * InsightFace Face Detection → face boxes within person crops.
    * Embedding Extraction → 512‑D face vectors.
    * Cosine Similarity (query vs. candidates) → ranked list.
5. Fallback Logic: If YOLOv8 or face detection fails, return the original image region; if a model errors, switch to a lighter backup.
6. Response: Ranked matches with scores, thumbnails, and provenance links.
Client → FastAPI → Background Job Queue → Downloader(s) → Detection → Embeddings → Similarity → Results Store → API Response

## Detailed Processing Flow
1. Upload & Validation
    * Validate mime/type and size, normalize color spaces, and de‑duplicate via perceptual hash.
2. Candidate Discovery & Download
    * Multi‑engine search strategies; retries with exponential backoff; per‑host rate limiting.
3. Preprocessing
    * Resize/pad to model input; EXIF orientation handling.
4. Detection & Embeddings
    * YOLOv8 identifies person regions (confidence threshold & NMS tuned).
    * InsightFace (retinaface/antelope variants) localizes faces and generates face embeddings.
5. Similarity & Ranking
    * L2‑normalized vectors; cosine similarity for ranking; top‑K selection with tie‑breakers (sharpness, face size, source quality).
6. Packaging Results
    * Return JSON with match score, source URL, and cropped visualization; persist artifacts per‑request directory.

## Fallback & Resilience
* Detector Failure:
    * If person detection fails → try face detection directly on full frame.
    * If face detection fails → return full image (with a flag fallback=true).
* Model Fallbacks:
    * Switch to a lighter/older checkpoint when GPU memory is constrained or inference errors occur.
* Network Fallbacks:
    * Alternate downloaders; cached proxies; timeouts and safe aborts.
* Graceful Degradation:
    * Always return best‑effort results with clear status codes and diagnostics.

## Concurrency & Performance
* Threading & Semaphores: Control max concurrent downloads/inferences to protect CPU/GPU and external services.
* Fire‑and‑Forget Jobs: Long‑running stages dispatched as background tasks so the API stays responsive.
* Batching & Warmup: Optional embedding batching and model warmup to reduce cold‑start latency.
* Caching: Reuse embeddings for repeated candidates; disk + in‑memory caches keyed by content hash.

## FastAPI Surface (Sample)
* POST /match — upload an image, returns a task id immediately (async) or synchronous result for small jobs.
* GET /match/{task_id} — retrieve results and logs.
* GET /healthz — liveness/readiness checks.
Response (simplified):
{
  "task_id": "c5a0...",
  "status": "completed",
  "query": {"faces": 1},
  "matches": [
    {"url": "https://...", "score": 0.92, "fallback": false, "thumb": "s3://.../crop.jpg"},
    {"url": "https://...", "score": 0.88, "fallback": false}
  ],
  "meta": {"elapsed_ms": 1840, "engine": "insightface_r100", "similarity": "cosine"}
}

## Deployment & Ops (AWS)
* Compute: Containerized services on ECS/EKS or EC2 with GPU instances where available.
* Storage: S3 for artifacts; lifecycle policies for cost control.
* Networking: Private subnets, NAT for egress, security groups locked to necessary ports.
* Monitoring: CloudWatch metrics + logs; alarms on error rates, latency, and GPU mem.
* CI/CD: Image builds, vulnerability scans, and blue/green deployments.

## Security & Privacy
* Explicit user consent for web retrieval.
* PII‑aware logging (hash or redact URLs and filenames).
* Signed URLs for artifact access; short TTLs.
* Configurable retention and right‑to‑erasure endpoints.

## Tech Stack
* Backend: Python, FastAPI
* Models: YOLOv8 (person), InsightFace (face), cosine similarity for ranking
* Infra: AWS (ECS/EKS/EC2), S3, CloudWatch
* Data: Local disk cache + S3, content‑hash indexing

## Results & Metrics (Illustrative)
* Median end‑to‑end latency: ~1.8s for 20 candidates on a T4 GPU
* Top‑1 precision@10: ~0.91 (internal validation set)
* Download success rate: > 98% with retry policies
Note: Replace with your live numbers; add a small table or chart if desired.

## Challenges & Lessons
* Handling varied image qualities and occlusions required tuned thresholds and prefilters.
* Concurrency without overloading GPUs demanded semaphore‑guarded pools.
* Source diversity (CDNs, throttling) necessitated resilient downloaders.

## Roadmap
* Add CLIP‑based re‑ranking for context cues.
* Integrate vector DB (FAISS/pgvector) for faster large‑scale retrieval.
* Progressive result streaming and UI gallery.
* Per‑region legal/compliance toggles.

## How to Demo
1. Upload a portrait to /match.
2. Poll /match/{task_id} or use the synchronous flag.
3. Inspect returned matches, scores, and crop overlays.
4. Review logs/metrics from CloudWatch.

## Credits
* Models: Ultralytics YOLOv8, InsightFace.
* Infra: AWS.

## Appendices
* Directory Layout
```python
FaceMatchingV1/  
├── core/                      # Main pipeline logic  
├── detectors/                 # Detection scripts (person/face)  
├── extractor/                 # Feature extraction utilities  
├── matcher/                   # Face comparison module  
├── utils/                     # Helper functions  
├── input/                     # Input images (auto-generated)
└── <request_id>/ 
│   ├── query/                 # Query images  
│   └── pool/                  # Pool of reference images  
├── output/                    # Results (auto-generated)
└── <request_id>/
│   ├── query/                 # Processed query outputs  
│   │   ├── annotated/         # Images with detection annotations  
│   │   ├── cropped_face/      # Extracted face crops  
│   │   ├── cropped_person/    # Extracted full-body crops  
│   │   └── embeddings.json/      # Face embeddings (vector data)  
│   ├── pool/                     # Processed pool outputs  
│   │   ├── annotated/            # Annotated pool images  
│   │   ├── cropped_face/         # Pool face crops  
│   │   ├── cropped_person/       # Pool full-body crops  
│   │   └── embeddings.json/      # Pool embeddings  
│   └── matching_results.json/    # Final face-matching results  
└── README.md                     # Documentation  
```

* Env Flags
    * MAX_CONCURRENCY, SIM_THRESHOLD, DOWNLOAD_TIMEOUT, FALLBACK_MODEL
