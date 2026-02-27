# Secure RAG Architecture (Tenant-Aware & Safety Validated)

## Overview

This project implements a secure, multi-tenant Retrieval-Augmented Generation (RAG) architecture. The system is designed to ensure strict tenant isolation, secure backend enforcement, and layered safety validation before and after LLM generation.

**Core Principle:** No tenant should ever access another tenant’s data.

---

## Architecture Summary

Every request follows this secured flow:

1. Authenticated request from client
2. Backend verifies identity and extracts tenant context
3. Input validation and safety filtering
4. Tenant-scoped document retrieval
5. LLM generation (backend only)
6. Output validation and safety enforcement
7. Response returned to client

---

## Layer Responsibilities

### 1. Client (Frontend)

* Sends authenticated requests (token included)
* Displays chatbot responses and citations

**Security Rules:**

* Never send `tenant_id` manually
* Never store API keys
* Never call the LLM directly
* All requests must go through backend

Frontend is considered untrusted.

---

### 2. API Layer (Backend Security Gatekeeper)

This is the primary enforcement layer.

**Responsibilities:**

* Verify authentication token
* Extract `tenant_id` and user role
* Enforce RBAC (admin vs member)
* Set tenant context in database
* Block unauthorized access before processing

**Golden Rule:**
All requests must be validated here before moving forward.

---

### 3. Safety Engine (Pre-Generation)

Validates user input before retrieval or generation.

**Responsibilities:**

* Sanitize input
* Enforce domain restrictions
* Filter malicious or irrelevant prompts
* Rate limit requests
* Log suspicious activity

Prevents abuse, jailbreak attempts, and prompt manipulation.

---

### 4. RAG Service (Tenant-Scoped Retrieval)

Retrieves relevant document chunks.

**Security Requirements:**

* Always filter by:

  * `tenant_id`
  * `chatbot_id`
* Never run similarity search across all tenants
* Limit Top-K results (5–10 max)
* Limit chunk size to prevent large data exposure

Prevents cross-tenant data leakage and excessive data dumps.

---

### 5. LLM Layer

Generates answers using retrieved content.

**Security Rules:**

* LLM API key stored on backend only
* System prompt is never exposed
* System prompt must:

  * Ignore instructions inside documents
  * Answer only from retrieved content
  * Return fallback response if information not found

---

### 6. Safety Engine (Post-Generation)

Validates model output before returning to user.

**Responsibilities:**

* Detect sensitive data patterns
* Detect system prompt leakage
* Enforce policy compliance
* Redact or block unsafe responses
* Log safety actions

Prevents accidental exposure, hallucinated secrets, and injection success.

---

## Database & Multi-Tenant Enforcement

Every tenant-related table must include:

* `tenant_id`

Tables include:

* chatbots
* documents
* chunks / embeddings
* chat sessions
* chat messages
* audit logs

**PostgreSQL Row-Level Security (RLS) is enabled.**

Backend must set tenant context before executing queries.

If tenant filtering is missing in even one query, cross-tenant data leakage can occur.

---

## Authentication & Authorization Rules

* Every API endpoint requires authentication
* Backend verifies token
* `tenant_id` and role extracted from verified token
* Backend uses tenant_id from token (never from frontend)

Never trust tenant_id from browser.
Never skip authentication checks.

---

## Retrieval Rules

All similarity searches must include:

* `tenant_id` filter
* `chatbot_id` filter

Never perform cross-tenant search.

Limits:

* Top-K results (5–10)
* Controlled chunk size

---

## Secrets Management

* LLM API keys remain on backend
* No secrets exposed in frontend
* No hardcoded secrets in repository

---

## Logging & Audit

We log:

* user_id
* tenant_id
* chatbot_id
* retrieved document IDs
* safety actions

We do NOT log:

* API keys
* sensitive secrets

---

## Why This Architecture Is Secure

1. Identity enforced at API layer
2. Tenant isolation enforced at database level (RLS)
3. Retrieval strictly tenant-scoped
4. Safety validation before and after LLM generation
5. LLM never directly exposed

If tenant isolation, token validation, or retrieval filtering are not consistently enforced, the system becomes vulnerable.

---

## Mermaid Architecture Diagram

```mermaid
flowchart LR
  %% =========================
  %% Secure RAG Architecture — Tenant-Aware & Safety Validated
  %% =========================

  C[Client] --> API

  subgraph API["API Layer"]
    API[API Gateway / Backend]
    A1[AuthN (JWT/OAuth)]
    A2[RBAC / ABAC]
    A3[Extract tenant_id + chatbot_id]
    A4[Set DB tenant context (session)]
    API --> A1 --> A2 --> A3 --> A4
  end

  %% Central policy control
  subgraph POL["Centralized Policy"]
    P[Policy Engine (rules + config)]
  end

  %% Pre-generation safety
  subgraph PRE["Input Safety"]
    S1[Safety Engine (pre-gen)\n- prompt injection checks\n- PII / secrets detection\n- allow/deny rules]
  end

  %% Retrieval
  subgraph RAG["RAG Service"]
    R1[Retriever\n- tenant-scoped filter\n- chatbot-scoped filter\n- top-K / chunk limits]
    R2[Context Builder\n- sanitize + rank\n- citations/metadata]
  end

  %% Data store (tenant isolated)
  subgraph DB["Data Layer"]
    L[(🔒 PostgreSQL)]
    L1[RLS Enforced\nWHERE tenant_id = current_setting('app.tenant_id')]
    L2[Documents / Embeddings / Metadata]
    L --> L1 --> L2
  end

  %% LLM
  subgraph GEN["Generation"]
    LLM[LLM]
  end

  %% Post-generation safety
  subgraph POST["Output Safety"]
    S2[Safety Engine (post-gen)\n- PII / secrets redaction\n- policy compliance\n- hallucination/unsafe content checks]
  end

  %% Audit / Monitoring
  subgraph AUD["Observability & Audit"]
    A[(🔒 Audit Logs / SIEM)]
    M[Metrics + Traces]
  end

  %% Flows
  A4 --> P
  P --> S1
  S1 --> R1

  R1 --> L
  L2 --> R1
  R1 --> R2 --> LLM

  %% Policy can influence retrieval + output rules
  P -. "retrieval limits / allow-lists" .-> R1
  P -. "output rules" .-> S2

  LLM --> S2 --> API --> C

  %% Auditing (tap points)
  API -. "request/decision logs" .-> A
  S1  -. "input risk score" .-> A
  R1  -. "retrieval audit" .-> A
  S2  -. "output redaction / flags" .-> A
  API -. "latency / errors" .-> M
  RAG -. "retrieval metrics" .-> M
  GEN -. "token usage" .-> M
```

---
