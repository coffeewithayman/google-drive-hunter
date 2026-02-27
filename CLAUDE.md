# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Google Drive Hunter (internally "Cool Drive Audit") is a Python tool that audits Google Workspace domains for publicly shared Google Drive files and optionally remediates them.

## Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Configure settings
cp settings.example settings.py
# Edit settings.py: set DOMAIN, ADMIN_USERNAME, SERVICE_ACCOUNT_FILE, LOCKDOWN_GRACE_DAYS
```

`settings.py` is gitignored (it holds domain-specific config). The service account JSON file (credentials) is also gitignored.

## Running the Tools

```bash
# Audit all publicly shared files across the domain
python3 audit.py

# Audit with specific output fields
python3 audit.py -f name link id modified

# Audit shared drives only
python3 audit.py --shared-drives-only

# Generate a Google Sheets report
python3 audit.py --sheets

# Console output only (skip HTML)
python3 audit.py --no-html

# Debug mode (verbose API errors, stack traces)
python3 audit.py --debug

# Lockdown (remove public sharing) for a specific user
python3 lockdown.py user@yourdomain.com
```

## Architecture

The codebase has three Python files:

- **`common.py`** — Shared Google API authentication and data-fetching logic. Manages credential caching (`_credentials` dict), scope definitions, and all Google API calls (Admin Directory, Drive, Shared Drives). Key functions: `delegated_credentials()`, `get_domain_users()`, `get_publicly_shared_files()`, `get_shared_drives()`, `collect_paginated()`.

- **`audit.py`** — Main audit script. Iterates all domain users and shared drives, finds publicly shared files, and outputs results as console text, per-user HTML files (in `out-YYYYMMDD-HHMMSS/`), and/or a Google Sheets report. Validates required APIs are enabled before running.

- **`lockdown.py`** — Remediation script for a single user. Finds that user's publicly shared files older than `LOCKDOWN_GRACE_DAYS`, then replaces `anyoneWithLink` permissions with domain-restricted permissions (same role, but restricted to the org domain). Interactive dry-run confirmation before committing changes. Logs changes to `out-ld-[email]-[timestamp].tsv`.

**Authentication flow:** Service account JSON → `google.oauth2.service_account.Credentials` → `.with_subject(email)` for domain-wide delegation. The `SCOPES` dict in `common.py` maps credential categories (`directory`, `audit`, `lockdown`, `sheets`) to the minimum required OAuth scopes.

**Public file detection:** Files are found using Google Drive API visibility filters `anyoneWithLink` and `anyoneCanFind` (constants `PUBLIC_PERMISSION_ID` / `DISCOVERABLE_PERMISSION_ID` in `common.py`).

## Required Google Cloud Configuration

- Service account with domain-wide delegation
- OAuth scopes: `admin.directory.user.readonly`, `drive.metadata.readonly`, `drive` (for lockdown), `spreadsheets`, `drive.file`
- APIs enabled: Admin SDK, Google Drive API, Google Sheets API (only if using `--sheets`)

The audit script validates API availability on startup and prints direct console links to enable missing APIs.
