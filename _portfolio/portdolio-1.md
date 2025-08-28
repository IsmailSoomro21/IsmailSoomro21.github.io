---
title: "Face Matching Engine Service"
excerpt: "AI-powered tool to match faces and retrieve similar images from the internet. <br/><img src='/images/proj-2.png'>" 
collection: portfolio
---

## Introduction 
A production‑ready system that finds visually similar faces on the web from a single uploaded photo, then ranks results using learned embeddings with cosine similarity. Built for speed, resilience, and scale with FastAPI and a robust fallback pipeline. This platform also expands the ecosystem, enabling journalists and fans to effortlessly discover and track their favorite personalities.


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
    * Embedding Extraction → 256-D face vectors.
    * Cosine Similarity (query vs. candidates) → ranked list.
5. Fallback Logic: If YOLOv8 or face detection fails, return the original image region; if a model errors, switch to a lighter backup.
6. Response: Ranked matches with scores, thumbnails, and provenance links.
<div style="display:flex;flex-wrap:wrap;align-items:center;gap:10px;font-family:sans-serif;">
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Client</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">FastAPI</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Trigger Query</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Image Download</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Job Queue</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Person Detection</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Face Detection</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Embeddings</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">Similarity Search</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">JSON Result Store</div> →
  <div style="padding:10px 15px;border:1px solid #ccc;border-radius:8px;background:#f9f9f9;">API Response</div>
</div>


## Detailed Processing Flow
1. Upload & Validation
    * Validate mime/type and size, normalize color spaces, and de‑duplicate via perceptual hash.
2. Candidate Discovery & Download
    * Multi‑engine search strategies; retries with exponential backoff; per‑host rate limiting.
3. Preprocessing
    * Resize/pad to model input; EXIF orientation handling.
4. Detection & Embeddings
    * YOLOv8 identifies person regions (confidence threshold, iou_threshold, dimensions & Blur tuned).
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
* POST /Process-s3-image — upload an image, returns a task id immediately (async) or synchronous result for small jobs.
* GET /status/{request_id} — retrieve results and logs.
* GET /status - Give overall status of all the requests failed, completed pending, queued 
* GET /health — liveness/readiness checks.

Response (simplified):

```python	
{
  "request_id": "9d4e7261",
  "status": "completed",
  "input_working_dir": "input/9d4e7261",
  "output_working_dir": "output/9d4e7261",
  "error": null,
  "retry_count": 0,
  "max_retries": 3,
  "time_taken": 69.27256917953491,
  "result": [
    {
      "query_image": "query-caaf1a9f.jpg",
      "query_uri": "s3://face-engine-bucket/Lionel_Messi_20180626 (1).jpg",
      "matches": [
        {
          "pool_image": "pool-4.jpg",
          "image_uri": "s3://face-engine-pool-bucket/136054219.jpg.0.jpg",
          "similarity": 0.5416,
          "match_status": "MATCH"
        }
      ]
    },
    {
      "query_image": "query-caaf1a9f.jpg",
      "query_uri": "s3://face-engine-bucket/Lionel_Messi_20180626 (1).jpg",
      "matches": [
        {
          "pool_image": "pool-43.jpg",
          "image_uri": "s3://face-engine-pool-bucket/skysports-messi-lionel-barcelona_5006546.jpg",
          "similarity": 0.5594,
          "match_status": "MATCH"
        }
      ]
    },
  ]
}
```

## Deployment & Ops (AWS)
* Compute: Containerized services on ECS Farget.
* Storage: S3 for artifacts; lifecycle policies for cost control.
* Networking: Private subnets, NAT for egress, security groups locked to necessary ports.
* Monitoring: CloudWatch metrics + logs; alarms on error rates, latency.

<!-- ## Security & Privacy
* Explicit user consent for web retrieval.
* PII‑aware logging (hash or redact URLs and filenames).
* Signed URLs for artifact access; short TTLs.
* Configurable retention and right‑to‑erasure endpoints. -->

## Tech Stack
* Backend: Python, FastAPI, OOP
* Models: YOLOv8 (person), InsightFace (face), cosine similarity for ranking
* Infra: AWS (ECS/ Farget), S3, CloudWatch
* Data: Local disk cache + S3

## Results & Metrics (Illustrative)
* Median end‑to‑end latency: ~1.8s for 20 candidates on a T4 GPU
* Top‑1 precision@10: ~0.91 (internal validation set)
* Download success rate: > 98% with retry policies

<!-- 
## Roadmap
* Add CLIP‑based re‑ranking for context cues.
* Integrate vector DB (FAISS/pgvector) for faster large‑scale retrieval.
* Progressive result streaming and UI gallery.
* Per‑region legal/compliance toggles. -->

<!-- ## How to Demo
1. Upload a portrait to /match.
2. Poll /match/{task_id} or use the synchronous flag.
3. Inspect returned matches, scores, and crop overlays.
4. Review logs/metrics from CloudWatch. -->

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
    * MAX_CONCURRENCY, DOWNLOAD_TIMEOUT, FALLBACK_MODEL



