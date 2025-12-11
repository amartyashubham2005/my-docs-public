---
title: Konetou Documentation
---

# Konetou Documentation

Welcome to the official documentation for **Konetou** - an intelligent RAG (Retrieval-Augmented Generation) system developed by R-Mad Ltd.

## Overview

Konetou is a production-ready RAG system that enables intelligent knowledge management by:

- **Automated Web Scraping** - Schedule periodic URL scanning with configurable intervals
- **Vector Search** - ChromaDB-powered semantic similarity search
- **Dual LLM Support** - Seamlessly switch between OpenAI (cloud) and Local (Ollama)
- **Multi-Tier Caching** - 3-level caching system for 3000x performance improvement
- **Streaming Responses** - Real-time response streaming via Server-Sent Events

## Quick Links

| Document | Description |
|----------|-------------|
| [Architecture](ARCHITECTURE.md) | Complete system architecture and design documentation |

## Technology Stack

| Layer | Technology |
|-------|------------|
| **Backend** | FastAPI, Python 3.9+ |
| **Vector DB** | ChromaDB |
| **Frontend** | React 19, TailwindCSS |
| **LLM (Cloud)** | OpenAI GPT-4o-mini |
| **LLM (Local)** | Ollama (llama3.2) |
| **Deployment** | Nginx, Systemd, GitHub Actions |

## Getting Started

Refer to the [Architecture Document](ARCHITECTURE.md) for detailed information about:

- System components and their interactions
- Data flow and RAG pipeline design
- API endpoints and usage
- Deployment and scaling strategies

---

**Konetou by R-Mad Ltd** - Intelligent RAG System for Modern Applications
