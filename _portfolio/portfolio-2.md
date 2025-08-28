---
title: "AI Juggling Detection Serivce"
excerpt: "Real-time multi-stream juggling detection<br/><img src='/images/proj-1.png' >"
collection: portfolio
---

## Introduction
A high-performance FastAPI-based microservice that detects and counts juggling patterns in videos. Supports both direct video uploads and S3-based processing, with robust error handling, monitoring, and asynchronous task management. Built for scalability and reliability, this service demonstrates expertise in video processing, cloud integration, and API development, making it suitable for real-world applications in sports analytics, entertainment, and automated video analysis.

## Features
* Video Processing
    * Accepts direct video uploads or S3 video URLs.
    * Detects and counts complex juggling patterns automatically.

* Asynchronous Task Management
    * Non-blocking processing for large video files.
    * Track request status in real-time.

* S3 Integration
    * Read/write videos to AWS S3 seamlessly.
    * Configurable input/output buckets.

* Configurable Resource Limits
    * Adjustable timeouts and maximum file sizes.
    * Automatic cleanup of temporary files to conserve resources.

* Monitoring & Logging
    * Health check endpoints for uptime monitoring.
    * Detailed logs for requests, responses, and errors.
    * Metrics for processing time and task performance.

## Error Handling
* Network & Processing Errors
    * Retry logic for network failures.
    * Timeout handling for long-running tasks.
    * Meaningful error messages for end users.

* HTTP Status Codes
    * 200 OK – Request processed successfully
    * 400 Bad Request – Invalid input or file format
    * 404 Not Found – Resource not found
    * 408 Request Timeout – Processing exceeded allowed time
    * 413 Payload Too Large – Video exceeds size limit
    * 500 Internal Server Error – Unexpected server error

## Best Practices Implemented
1. State Management
    * Request ID generation and status tracking.
    * Offline resilience and caching for faster repeated requests.

2. User Experience
    * Progress updates and loading indicators.
    * Retry options for failed uploads or processing tasks.

3. Security
    * HTTPS enforced for all API endpoints.
    * Secure API key handling.
    * Input validation to prevent malformed or unsafe requests.

## Monitoring & Observability
* Health check endpoint for service uptime monitoring.
* Request/response tracking for debugging and analytics.
* Metrics on processing times and task completion rates.

## Technical Stack
* Backend: FastAPI, Python 3.11+
* Video Processing: YOLOv11 
* Async Processing: asyncio
* Cloud Storage: AWS S3
* Logging & Monitoring: Python logging


## Output Demo Link
* https://drive.google.com/file/d/1_8EoOugfJ8-ROnChKjtq9blb-o6epvPL/view?usp=drive_link

## Service Demo Link
* https://drive.google.com/file/d/1msqynL_ZfO3MOXqy396FJ1hRh4-h1oGG/view?usp=drive_link