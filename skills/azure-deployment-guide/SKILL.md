---
skill: azure-deployment
version: 1.0.0
context: current
description: |
  Guided walkthrough for deploying applications to Azure Container Apps following
  proven production patterns. Covers managed identity setup, secrets management,
  OIDC authentication, blue-green deployments, and comprehensive smoke testing.
tags: [azure, deployment, container-apps, production]
---

# Azure Deployment Guide

You are guiding a developer through deploying an application to Azure Container Apps
following proven production patterns.

## Prerequisites Check

Before starting, verify:

```bash
# Azure CLI authenticated
az account show

# Correct subscription selected
az account list --output table

# Docker installed
docker --version

# GitHub CLI (for OIDC setup)
gh --version
```

If any are missing, guide the developer to install them first.

---

## Overview

This guide walks through creating a complete Azure deployment following 7 proven patterns:

1. **Managed Identity for ACR Pull** — No credential storage
2. **Secrets vs Environment Variables** — Sensitive data segregation
3. **OIDC Authentication** — GitHub Actions without passwords
4. **Application Insights** — Comprehensive telemetry
5. **Blue-Green Deployment** — Zero-downtime releases
6. **Comprehensive Smoke Tests** — 5-stage verification
7. **GitHub Secrets Management** — Secure credential workflow

---

## Stage 1: Infrastructure Setup

Walk the developer through creating:
- Resource Group
- Container Registry (ACR)
- Log Analytics Workspace
- Container Apps Environment
- Application Insights

For detailed commands, reference: `@dev-ops:context/azure-deployment-patterns.md`

## Stage 2: Container App with Managed Identity

Guide them to:
- Create Container App with placeholder image
- Enable system-assigned managed identity
- Grant AcrPull role to ACR
- Configure registry authentication via managed identity

## Stage 3: Secrets Management

Show them how to:
- Get Application Insights connection string
- Set Container App secrets (db-password, api-keys, telemetry-connection-string)
- Reference secrets in environment variables with `secretref:` syntax

## Stage 4: GitHub OIDC Setup

Walk through:
- Creating Azure AD app registration
- Creating service principal
- Setting up federated credential for GitHub repo
- Assigning Contributor role to resource group
- Saving required secrets in GitHub

## Stage 5: GitHub Workflow

Guide them to create `.github/workflows/deploy.yml` with:
- OIDC authentication
- Docker build and push to ACR
- Blue-green deployment (0% → smoke tests → 100%)
- Comprehensive smoke testing

## Stage 6: First Deployment

Help them:
- Commit and push the workflow
- Monitor the workflow run
- Troubleshoot any failures
- Verify the deployment

---

## Quick Troubleshooting

Common issues and solutions are in `@dev-ops:context/azure-deployment-patterns.md` under "Common Commands" and "Anti-Patterns".

Delegate complex Azure questions to `azure-zap:azure-mcp-expert` for authoritative guidance.
