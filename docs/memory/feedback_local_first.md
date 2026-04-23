---
name: Local-first Docker Compose dev before cloud
description: Geoffrey wants CORPUS developed and working on Docker Compose on a Linux machine first; Azure deployment is its own later sub-project
type: feedback
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
For CORPUS, **develop and ship locally on Docker Compose first**; Azure deployment is a separate later sub-project — don't mix cloud-native infra into early specs.

**Why:** Geoffrey corrected an early cloud-native-from-day-one architecture. He wants "a working tool on standard linux tools, in a machine" before reaching for Azure Container Apps, Key Vault, Application Insights, Managed Identity, etc. Keeps the feedback loop tight, defers cloud-billing/operational complexity, and because the app uses twelve-factor patterns, the eventual Azure deploy is a mechanical port rather than a rewrite.

**How to apply:**
- Early specs and implementation plans target Docker Compose on a Linux host.
- Env-var config, JSON stdout logs, stateless containers, migrations at entrypoint — all twelve-factor from day one.
- Azure specifics (Container Apps, Azure Database for PostgreSQL, Key Vault, App Insights, Managed Identity, VNet, Entra ID) get their own "Deploy to Azure" sub-project spec later.
- Don't bake cloud-specific SDKs into core modules; abstract via interfaces (e.g., `ObjectStore` with Filesystem + AzureBlob implementations).
