# PulseIQ

### Turn product signals into intelligent insights.

PulseIQ is an AI-powered product analytics platform that transforms raw event data into actionable insights, user behavior analysis, and intelligent recommendations.

Instead of manually creating SQL queries or dashboards, users can upload datasets and instantly understand:

- What happened
- Why it happened
- What should happen next

---

## Overview

PulseIQ helps teams understand product health and user behavior through analytics and AI.

Upload event data and automatically generate:

- User activity metrics
- Retention analysis
- Funnel analysis
- Cohort analysis
- Revenue insights
- Pattern detection
- AI-powered recommendations

---

## Core Workflow

```text
Upload Dataset
      ↓
Schema Detection
      ↓
Validation
      ↓
Normalization
      ↓
Metrics Calculation
      ↓
Pattern Detection
      ↓
AI Insight Generation
      ↓
Dashboard & Reports
```

---

## Features

### MVP — Core Analytics Foundation

**Dataset Processing**

- CSV upload
- Automatic schema detection
- Manual field mapping
- Validation & normalization
- Dataset history

**Analytics**

- DAU / WAU / MAU
- New & returning users
- D1 / D7 / D30 retention

**Dashboard**

- User growth charts
- Retention charts
- Metrics overview

**AI Insights**

- Key findings
- Trend analysis
- Recommended actions

---

### V1 — Advanced Analytics

**Behavior Analysis**

- Funnel analysis
- Cohort analysis
- User segmentation

**Business Metrics**

- Revenue analytics
- Conversion metrics
- ARPU

**Reporting**

- PDF export
- CSV export
- Scheduled reports

---

### V2 — AI Analyst

**Natural Language Analytics**

- Ask questions in plain language
- AI-generated queries
- Insight summaries

**Intelligence Features**

- Weekly summaries
- Insight ranking
- Anomaly detection

---

### V3 — Realtime Platform

**Data Ingestion**

- API support
- SDK integration
- Webhooks

**Exploration**

- Event explorer
- Event timeline
- Search & filtering

**Collaboration**

- Workspaces
- Team members
- Roles & permissions

---

### Future Vision — AI Data Intelligence Platform

**Predictive Intelligence**

- Root cause analysis
- Churn prediction
- LTV prediction
- Revenue forecasting

**Automation**

- Smart alerts
- AI-generated dashboards
- Experiment analysis

---

## Required Dataset Fields

Minimum fields required for analytics:

| Field      | Required |
| ---------- | -------- |
| user_id    | Yes      |
| timestamp  | Yes      |
| event_name | Yes      |

Optional fields:

- channel
- revenue
- country
- device
- campaign
- properties

---

## Architecture

Architecture patterns:

- Clean Architecture
- Domain-Driven Design (DDD)
- Hexagonal Architecture
- CQRS
- Domain Events
- Modular Monolith → Microservices migration path

---

## Project Structure

```text
src/

├── modules
│   ├── dataset
│   ├── analytics
│   ├── insight
│   ├── reports
│   └── users
│
├── shared
│   ├── kernel
│   ├── common
│   └── events
│
└── infrastructure
```

---

## Event Workflow

```text
DatasetUploaded
      ↓
SchemaValidated
      ↓
DataNormalized
      ↓
MetricsCalculated
      ↓
InsightsGenerated
```

---

## Tech Stack

### Frontend

- Next.js
- TypeScript
- Tailwind CSS
- Recharts

### Backend

- NestJS
- TypeScript

### Storage

- PostgreSQL
- MinIO

### Queue

- Redis
- BullMQ

### Infrastructure

- Docker

### Future Scaling

- ClickHouse
- Kubernetes

### AI

- LLM Integration

---

## Roadmap

**Phase 1**

- Upload datasets
- Core metrics
- Dashboard
- AI insights

**Phase 2**

- Funnel analysis
- Cohort analysis
- Segmentation

**Phase 3**

- AI analyst
- Natural language analytics

**Phase 4**

- Prediction engine
- Root cause analysis
- Full AI data platform

---

## Mission

Transform raw data into meaningful decision
