# NeuroLink Video Analysis Implementation Plan

**Provider: Google Cloud Video Intelligence**

## Overview

This document describes the design and implementation of Video Intelligence support in NeuroLink, built on top of the Google Cloud Video Intelligence API.

NeuroLink exposes a unified, provider-agnostic interface for analyzing video content, with support for speech transcription, visual understanding, content moderation, and structural video analysis.

## Table of Contents

1. [Problem Statement & Solution](#problem-statement--solution)
2. [Architecture Overview](#architecture-overview)
3. [Core Components](#core-components)
4. [NeuroLink SDK Integration](#neurolink-sdk-integration)
5. [CLI Integration](#cli-integration)
6. [Output Model (VideoIntelligenceResult)](#output-model-videointelligenceresult)
7. [Configuration](#configuration)
8. [Error Handling](#error-handling)
9. [Extensibility Roadmap](#extensibility-roadmap)
10. [Conclusion](#conclusion)

## Problem Statement & Solution

### Problem Statement

NeuroLink lacked a **unified Video Intelligence layer**, forcing developers to deal with provider-specific APIs, inconsistent outputs, and duplicated logic across SDKs and CLI tools. This made video workflows harder to build, extend, and maintain.

The **Google Cloud Video Intelligence API** analyzes video inputs provided either as inline content or via Cloud Storage, providing:
- **Speech transcription** (audio → text)
- **Label detection** (objects, activities, concepts)
- **Shot/scene detection** (structural analysis)
- **OCR** (extract in-video text)
- **Explicit content detection**

This enables intelligent video workflows beyond simple transcription—from automated summaries to content moderation.

### Solution

NeuroLink adds a provider-agnostic Video Intelligence pipeline:

* A single processor for validation, routing, and error handling
* Pluggable provider handlers for easy extensibility
* Normalized video intelligence results with feature-specific metadata (timestamps, confidence, etc.)
* Consistent SDK and CLI interfaces

This makes Video Intelligence a first-class, scalable capability within NeuroLink.

## Architecture Overview

The Video Intelligence flow follows the below pattern:

```text
Video File (Local / GCS)
        ↓
VideoIntelligenceProcessor
        ↓
VideoIntelligenceHandler (Google)
        ↓
Google Cloud Video Intelligence API
        ↓
VideoIntelligenceResult
```


### Design Principles

* Provider abstraction via a unified **Video Intelligence handler interface**
* Central orchestration through a **Video Intelligence processor**
* Thin, feature-focused provider adapters
* Shared validation, error handling, and logging across providers
* Consistent SDK and CLI ergonomics across video analysis workflows
* Explicit handling of asynchronous, long-running video analysis operations


## Core Components

1. **Video Intelligence Types (`videoIntelligenceTypes.ts`)**

   Defines all shared data contracts for Video Intelligence.

   Key responsibilities:

   * Supported video formats (e.g. MP4, MOV, AVI, MKV, subject to provider-supported codecs)
   * Video analysis configuration (`VideoIntelligenceOptions`)
   * Feature-specific configuration (speech transcription, labels, OCR, moderation, shots, etc.)
   * Feature-level metadata (timestamps, confidence scores, bounding boxes where applicable)
   * Final unified output model (`VideoIntelligenceResult`)

   These types are provider-agnostic and reused across the SDK, CLI, processors, and handlers.


2. **Video Intelligence Processor (`VideoIntelligenceProcessor`)**

   The central orchestrator for all Video Intelligence operations.

   Responsibilities:

   * Register Video Intelligence providers
   * Validate video input (format, empty buffer, size and duration limits)
   * Accept local video files directly or upload them to GCS when required for large or long-running analyses
   * Ensure provider availability and configuration
   * Delegate requested video intelligence features to the selected provider handler
   * Manage asynchronous polling and long-running operations
   * Normalize provider responses and error handling

   Why this exists:

   * Keeps provider-specific logic isolated
   * Allows runtime provider switching
   * Abstracts the asynchronous nature of Video Intelligence APIs


3. **Video Intelligence Handler Interface**

   All Video Intelligence providers implement a common handler interface.

   Core methods:

   * `analyze(video, options)`
   * `isConfigured()`

   Provider metadata includes:

   * Maximum supported video size
   * Maximum supported video duration
   * Supported video intelligence features

   This guarantees:

   * Predictable limits and pre-flight validation
   * Safe handling before invoking provider APIs
   * Uniform behavior across providers


4. **Google Video Intelligence Handler (`GoogleVideoIntelligenceHandler`)**

   Concrete implementation built on the **Google Cloud Video Intelligence API**.

   Supported capabilities (feature-dependent):

   * Speech transcription (punctuation, timestamps, alternatives)
   * Label detection
   * Face and person detection
   * Object tracking
   * Explicit content detection
   * Shot / scene change detection
   * Text detection (OCR)

   Notes:

   * All operations are asynchronous and handled via long-running annotations
   * Input videos may be provided either via inline content or GCS URIs; local files may be uploaded temporarily to GCS when required by size or duration constraints
   * Provider-enforced limits apply (video size, duration, and one video per request)

   This handler maps Google’s provider-specific responses into the unified
   `VideoIntelligenceResult` format.

## NeuroLink SDK Integration

### Public APIs

NeuroLink exposes high-level **Video Intelligence** APIs for analyzing video content:

* analyzeVideo(videoPath, options)
* analyzeVideoBuffer(videoBuffer, options)

These APIs:

* Automatically route requests to the configured provider
* Handle direct local video inputs or GCS uploads when required
* Manage asynchronous, long-running analysis operations
* Return a unified, normalized `VideoIntelligenceResult`
* Provide consistent SDK ergonomics across video analysis features


## CLI Integration

A new CLI command is added:

```bash
neurolink analyze-video <video-file>
```

### Supported CLI Features

* Provider selection
* Feature selection (speech transcription, labels, OCR, content moderation, shot detection, etc.)
* Language configuration for speech transcription (e.g. `--language en-US`)
* Automatic punctuation for transcription (default: true)
* JSON or text output (feature-dependent)
* Progress indication for asynchronous, long-running video analysis operations

### Example

```bash
neurolink analyze-video meeting-recording.mp4 \
  --features speech,labels,shots \
  --language en-US \
  --enable-punctuation \
  -o results.json
```



## Output Model (VideoIntelligenceResult)

All providers return a unified, normalized output structure that may include results from multiple video intelligence capabilities, depending on the requested features.

Includes (where applicable):

* Speech transcription results
  * Transcript text (concatenated or segmented)
  * Alternative transcriptions (if requested)
  * Word-level timing and confidence scores
  * Optional speaker tags
* Visual understanding results
  * Labels, objects, faces, persons, and logos with associated timestamps
* Content moderation signals
  * Explicit or sensitive content annotations with time offsets
* Structural analysis
  * Shot or scene boundaries and video segments
* Video metadata
  * Video duration
  * Feature-level processing details
* Provider metadata
  * Operation identifiers
  * Processing latency
  * Model or feature versions used (where available)

This unified output model ensures:

* Stable SDK contracts across providers and features
* Easy downstream processing (e.g. subtitles, search indexing, moderation pipelines)
* Provider-independent consumption of video intelligence results

## Configuration

### Google Cloud Setup

Enable the API:

gcloud services enable videointelligence.googleapis.com


Authenticate using one of:

Recommended (Service Account)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"


or environment-based credentials (Vertex AI / GCP runtime).

Additional setup:

* GCS bucket access (for input videos & temporary uploads)
* Sufficient quota for long-running operations

## Error Handling

Video Intelligence errors use the same NeuroLink error system:

* Structured error codes (e.g. VIDEO_TOO_LONG, GCS_UPLOAD_FAILED)
* Severity levels for classification and reporting
* Retriable vs non-retriable error categorization (for example, rate limits are retriable)
* Provider-specific context such as operation identifiers and partial failures

This enables:

* Consistent and predictable CLI output
* Uniform SDK exceptions across video analysis features
* Improved observability, debugging, and operational insight


## Extensibility Roadmap

This design establishes a flexible foundation for **Video Intelligence** in NeuroLink and is currently implemented using the **Google Cloud Video Intelligence API**. It is intentionally built to support future extensions, including:

* Improved speaker diarization through provider support or post-processing pipelines
* Near-real-time or streaming-style video analysis via providers or architectures that support it (not currently supported by Google Video Intelligence)
* Additional providers such as AWS Transcribe (video), Azure Video Indexer, and OpenAI Whisper (via video-to-audio extraction)
* Combining multiple video intelligence capabilities (speech transcription, labels, shots, OCR, moderation) within a single analysis pass where supported

All future providers only need to implement the unified **Video Intelligence handler interface** to integrate seamlessly with NeuroLink.


## Conclusion

This Video Intelligence implementation:

* Introduces a provider-agnostic video analysis layer in NeuroLink
* Uses clean, extensible provider abstractions
* Transparently handles asynchronous, long-running video analysis operations
* Supports both SDK and CLI workflows with consistent interfaces
* Enables multiple video intelligence capabilities, including speech transcription, visual understanding, content moderation, and structural analysis

Result: Video Intelligence becomes a first-class capability within NeuroLink, providing a strong foundation for future providers and features.

