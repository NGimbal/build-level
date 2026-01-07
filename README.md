# Build Level

A construction estimate comparison tool built on Cloudflare's platform.

## Overview

This application allows users to:

1. Upload multiple PDF construction estimates
2. Automatically extract and parse line items using AI
3. Classify line items into a standardized cost code format
4. Compare estimates side-by-side with computed statistics
5. Manually reclassify line items to improve accuracy

## Tech Stack

| Layer          | Technology                                   |
| -------------- | -------------------------------------------- |
| Frontend       | Preact, TanStack Query, TanStack Table, Vite |
| Backend        | Cloudflare Workers, Hono                     |
| Database       | Cloudflare D1 (SQLite)                       |
| Storage        | Cloudflare R2                                |
| AI             | Cloudflare Workers AI (Llama 3.1), BAML      |
| PDF Processing | pdfjs-serverless                             |

## Project Structure

```
build-level/
├── packages/
│   ├── shared/          # Shared types and utilities
│   ├── backend/         # Cloudflare Worker API
│   └── frontend/        # Preact SPA
├── docs/                # Module documentation
├── package.json         # Workspace root
├── pnpm-workspace.yaml
└── README.md
```

## Quick Start

### Prerequisites

- Node.js 18+
- pnpm 8+
- Cloudflare account with Workers, D1, R2, and AI enabled
- Wrangler CLI (`npm install -g wrangler`)

### Setup

```bash
# Clone the repository
git clone <repo-url>
cd build-level

# Install dependencies
pnpm install

# Setup Cloudflare resources
cd packages/backend
wrangler d1 create build-level
wrangler r2 bucket create build-level-files

# Update wrangler.toml with your database ID

# Run database migrations
wrangler d1 execute build-level --file=./src/db/schema.sql
wrangler d1 execute build-level --file=./src/db/seed.sql

# Start development
pnpm dev
```

### Development

```bash
# Run all packages in dev mode
pnpm dev

# Run only backend
pnpm --filter backend dev

# Run only frontend
pnpm --filter frontend dev

# Type check
pnpm typecheck

# Build for production
pnpm build
```

## Module Documentation

Detailed documentation for each module:

- [01 - Project Setup](./docs/01-project-setup.md)
- [02 - Shared Package](./docs/02-shared-package.md)
- [03 - Database Layer](./docs/03-database-layer.md)
- [04 - PDF Extraction](./docs/04-pdf-extraction.md)
- [05 - Estimate Parsing](./docs/05-estimate-parsing.md)
- [06 - Classification](./docs/06-classification.md)
- [07 - Comparison Engine](./docs/07-comparison-engine.md)
- [08 - API Routes](./docs/08-api-routes.md)
- [09 - Frontend Core](./docs/09-frontend-core.md)
- [10 - Upload Flow](./docs/10-upload-flow.md)
- [11 - Comparison Table](./docs/11-comparison-table.md)
- [12 - Estimate Table](./docs/12-estimate-table.md)
- [13 - Deployment](./docs/13-deployment.md)
- [14 - BAML Integration](./docs/14-baml-integration.md)

## API Reference

See [API Documentation](./docs/api-reference.md) for full endpoint details.
