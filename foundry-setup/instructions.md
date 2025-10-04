---
title: Instructions
description: Feedforward Foundry
published: true
date: 2025-10-04T17:20:26.578Z
tags: 
editor: markdown
dateCreated: 2025-10-03T20:18:41.059Z
---

# Installation Instructions

The best way to deploy the Foundry depends on your organization's computing environment. However, all services that make up the Foundry can be built as Docker images.

These Docker images can be used with Docker Compose, a Kubernetes environment, or as standalone containers.

For testing and as an example deployment, this guide shows you how to get the Foundry running with Docker Compose on your local machine.

## Prerequisites

- Docker Engine 20.10 or later
- Docker Compose V2
- At least 2GB of available RAM
- At least 2 CPU cores (vCPU)
- An OpenAI API key

## Quick Start

1. **Clone the repository**
   ```bash
   git clone --recursive https://github.com/ff-foundry/example-deployment
   cd example-deployment
   ```

2. **Configure environment variables**
   ```bash
   cp .env.example .env
   ```

3. **Edit `.env` file** with your settings:
   - Set `APP_TITLE` to your desired application name
   - Add your `OPENAI_API_KEY` (required)
   - Optionally add other LLM provider API keys (Anthropic, Google, etc.)
   
4. **Start the Foundry**
   ```bash
   docker-compose -f docker-compose.yml -f docker-compose-local.yml up
   ```

5. **Access the application**
   - Open your browser to `http://localhost:3080`
   - Create an account (registration is enabled by default)

## Services Architecture

The Foundry consists of six interconnected services:

- **api**: Main Foundry application server, which is based on Librechat, listens on port 3080.
- **foundry_mcp**: MCP (Model Context Protocol) server for agent creation, listens on port 8123.
- **mongodb**: Database for storing conversations and user data
- **meilisearch**: Search engine for message indexing
- **vectordb**: PostgreSQL with pgvector for RAG (Retrieval-Augmented Generation)
- **rag_api**: API service for document processing and retrieval

## Configuration Files

### docker-compose.yml
Main orchestration file defining all services and their relationships. Uses named volumes for data persistence.

### .env
Contains all environment variables including:
- Application settings (APP_TITLE, DOMAIN_SERVER, DOMAIN_CLIENT)
- Security keys (must be changed for production)
- LLM provider API keys
- Feature flags

### librechat.yaml
Configures the MCP server connection for agent creation capabilities.

## Data Persistence

Data is stored in Docker named volumes:
- `mongodb_data`: User accounts and conversation history
- `meili_data`: Search indexes
- `pgdata2`: Vector embeddings for RAG
- `librechat_uploads`: Uploaded files
- `librechat_images`: User-uploaded images
- `librechat_logs`: Application logs

## Security Considerations

**Before deploying to production:**

1. Generate new secure random values for all keys in `.env`:
   - CREDS_KEY, CREDS_IV
   - JWT_SECRET, JWT_REFRESH_SECRET
   - MEILI_MASTER_KEY
   - MAC_KEY

   Use the [LibreChat credentials generator](https://www.librechat.ai/toolkit/creds_generator) or generate your own cryptographically secure random values.

2. Update `DOMAIN_SERVER` and `DOMAIN_CLIENT` to your actual domains. It's recommended you use a reverse proxy (e.g. nginx-proxy) or load balancer.
3. Consider disabling registration (`ALLOW_REGISTRATION=false`) and manually creating user accounts
4. Keep `BAN_VIOLATIONS=false` (required for MCP server functionality)
