---
title: Environment Variables
description: 
published: true
date: 2025-10-06T10:58:15.071Z
tags: 
editor: markdown
dateCreated: 2025-10-06T09:56:21.405Z
---

# Environment Variables
The Foundry environment provides a setup which will work without tuning for local, testing deployments. If you want to go further, you'll most likely need to adjust one or more of these environment variables.
Please also see ``.env.example``.
# Foundry setup specific / Shared
The following environment variables are shared between the Foundry API service, the Foundry MCP and the Exhaustor service.
| Name | Description | Default (if any) |
| -------- | ------- | ------- |
| MAC_KEY | Used to securely share data between the services<br>See .env.example for generation instructions. | - |
| OPENAI_API_KEY | OpenAI AI key, used by Foundry API, MCP, Exhaustor | - |
| ANTHROPIC_API_KEY | Anthropic API key, used by Foundry API (by default, Agents created by Foundry API/MCP use Claude) | - |
| COOKIE_INSECURE | For localhost deployments, don't send secure for cookies.<br>Omit in production | - |
| FOUNDRY_EXHAUSTOR_URL | The client visible URL to your exhaustor instance. | - |
# Exhaustor
| Name | Description | Default (if any) |
| -------- | ------- | ------- |
| SESSION_SECRET | Should be random entropy for protecting sessions<br>In the example deployment, we use a randomness already generated for the Foundry API service. | - |
# Foundry API
The Foundry API service is based on Librechat. It shares in common many tunables and environment variables with Librechat, here are some of the most important ones.
| Name | Description | Default (if any) |
| -------- | ------- | ------- |
| APP_TITLE | The name shown to users for the service. | LibreChat
| CREDS_KEY | 32-byte key (64 characters in hex) for securely storing credentials. Required for app startup. | - |
| CREDS_IV | 16-byte IV (32 characters in hex) for securely storing credentials. Required for app startup. | - |
| JWT_SECRET | JWT secret key. Used for authentication. | - |
| JWT_REFRESH_SECRET | JWT refresh secret key. | - |
| MEILI_MASTER_KEY | The master key for MeiliSearch.<br>This should also match in the MeiliSearch container. | - |