# Module 13: Deployment

## Overview

Production deployment of the Estimate Compare application using Cloudflare's platform. Includes Workers deployment, Pages hosting, D1 migration, and monitoring setup.

## Goals

- Deploy backend to Cloudflare Workers
- Deploy frontend to Cloudflare Pages
- Configure production D1 database
- Setup custom domain (optional)
- Configure monitoring and logging
- Establish CI/CD pipeline

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Cloudflare Edge                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐       ┌──────────────────────────────┐   │
│  │  Cloudflare Pages │       │    Cloudflare Workers        │   │
│  │   (Frontend)      │──────▶│      (API Backend)           │   │
│  │                   │       │                              │   │
│  │  - Static assets  │       │  - Hono routes               │   │
│  │  - Preact SPA     │       │  - Business logic            │   │
│  │  - CDN cached     │       │  - AI processing             │   │
│  └──────────────────┘       └──────────────────────────────┘   │
│                                        │                         │
│                              ┌─────────┴─────────┐              │
│                              ▼                   ▼              │
│                    ┌──────────────┐    ┌──────────────┐        │
│                    │  Cloudflare  │    │  Cloudflare  │        │
│                    │     D1       │    │     R2       │        │
│                    │  (Database)  │    │  (Storage)   │        │
│                    └──────────────┘    └──────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Cloudflare account with Workers Paid plan (for D1, R2, AI)
- Wrangler CLI installed and authenticated
- Domain configured in Cloudflare (optional)
- Git repository setup

## Backend Deployment

### 1. Configure wrangler.toml for Production

```toml
# packages/backend/wrangler.toml

name = "estimate-compare-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

# Production D1 Database
[[d1_databases]]
binding = "DB"
database_name = "estimate-compare"
database_id = "your-production-database-id"

# Production R2 Bucket
[[r2_buckets]]
binding = "R2"
bucket_name = "estimate-compare-files"

# Workers AI
[ai]
binding = "AI"

# Environment variables
[vars]
ENVIRONMENT = "production"
LOG_LEVEL = "info"

# Production routes (optional custom domain)
# routes = [
#   { pattern = "api.yourdomain.com/*", zone_name = "yourdomain.com" }
# ]

# Development environment
[env.dev]
name = "estimate-compare-api-dev"

[[env.dev.d1_databases]]
binding = "DB"
database_name = "estimate-compare-dev"
database_id = "your-dev-database-id"

[[env.dev.r2_buckets]]
binding = "R2"
bucket_name = "estimate-compare-files-dev"

[env.dev.vars]
ENVIRONMENT = "development"
LOG_LEVEL = "debug"

# Staging environment
[env.staging]
name = "estimate-compare-api-staging"

[[env.staging.d1_databases]]
binding = "DB"
database_name = "estimate-compare-staging"
database_id = "your-staging-database-id"

[[env.staging.r2_buckets]]
binding = "R2"
bucket_name = "estimate-compare-files-staging"

[env.staging.vars]
ENVIRONMENT = "staging"
LOG_LEVEL = "debug"
```

### 2. Create Production Resources

```bash
# Navigate to backend package
cd packages/backend

# Create production D1 database
wrangler d1 create estimate-compare
# Note the database ID and add to wrangler.toml

# Create production R2 bucket
wrangler r2 bucket create estimate-compare-files

# Create staging resources (optional)
wrangler d1 create estimate-compare-staging
wrangler r2 bucket create estimate-compare-files-staging
```

### 3. Run Database Migrations

```bash
# Production
wrangler d1 execute estimate-compare --file=./src/db/schema.sql
wrangler d1 execute estimate-compare --file=./src/db/seed.sql

# Staging (if using)
wrangler d1 execute estimate-compare-staging --env=staging --file=./src/db/schema.sql
wrangler d1 execute estimate-compare-staging --env=staging --file=./src/db/seed.sql

# Verify tables
wrangler d1 execute estimate-compare --command="SELECT name FROM sqlite_master WHERE type='table';"
```

### 4. Deploy Worker

```bash
# Deploy to production
wrangler deploy

# Deploy to staging
wrangler deploy --env=staging

# Verify deployment
curl https://estimate-compare-api.<your-subdomain>.workers.dev/api/health
```

## Frontend Deployment

### 1. Configure for Production

```typescript
// packages/frontend/vite.config.ts
import { defineConfig } from 'vite';
import preact from '@preact/preset-vite';

export default defineConfig(({ mode }) => ({
  plugins: [preact()],
  resolve: {
    alias: {
      'react': 'preact/compat',
      'react-dom': 'preact/compat',
      'react/jsx-runtime': 'preact/jsx-runtime',
    },
  },
  build: {
    outDir: 'dist',
    sourcemap: mode !== 'production',
    minify: 'terser',
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['preact', '@tanstack/react-query', '@tanstack/react-table'],
        },
      },
    },
  },
  define: {
    'import.meta.env.VITE_API_URL': JSON.stringify(
      mode === 'production'
        ? 'https://estimate-compare-api.<your-subdomain>.workers.dev/api'
        : '/api'
    ),
  },
}));
```

### 2. Create Cloudflare Pages Project

```bash
# Option 1: Via Wrangler
cd packages/frontend
npx wrangler pages project create estimate-compare

# Option 2: Via Dashboard
# Go to Cloudflare Dashboard > Pages > Create Project
```

### 3. Configure Build Settings

In Cloudflare Pages dashboard or wrangler.toml:

```toml
# packages/frontend/wrangler.toml (for Pages)
name = "estimate-compare"
pages_build_output_dir = "dist"

[build]
command = "pnpm build"

[build.environment]
NODE_VERSION = "18"
```

### 4. Deploy Frontend

```bash
# Build
cd packages/frontend
pnpm build

# Deploy via Wrangler
wrangler pages deploy dist --project-name=estimate-compare

# Or connect to Git for automatic deployments
```

### 5. Configure Redirects (for SPA)

```
# packages/frontend/public/_redirects
/*    /index.html   200
```

## Environment Configuration

### Production Environment Variables

```bash
# Set secrets via Wrangler (if needed)
wrangler secret put API_KEY
```

### Frontend Environment

```typescript
// packages/frontend/src/config.ts
export const config = {
  apiUrl: import.meta.env.VITE_API_URL || '/api',
  environment: import.meta.env.MODE,
};
```

## Custom Domain Setup

### Backend (Workers)

```bash
# Add custom domain route in wrangler.toml
# routes = [{ pattern = "api.yourdomain.com/*", zone_name = "yourdomain.com" }]

# Or via dashboard:
# Workers & Pages > Your Worker > Settings > Triggers > Custom Domains
```

### Frontend (Pages)

```bash
# Via dashboard:
# Pages > Your Project > Custom Domains > Add Custom Domain
# Add: app.yourdomain.com

# Configure DNS:
# CNAME app -> estimate-compare.pages.dev
```

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
  CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
          
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'
          
      - run: pnpm install
      - run: pnpm typecheck
      # - run: pnpm test (if tests exist)

  deploy-backend:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
          
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'
          
      - run: pnpm install
      
      - name: Deploy Worker
        run: |
          cd packages/backend
          npx wrangler deploy

  deploy-frontend:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
          
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'
          
      - run: pnpm install
      
      - name: Build Frontend
        run: |
          cd packages/frontend
          pnpm build
          
      - name: Deploy to Pages
        run: |
          cd packages/frontend
          npx wrangler pages deploy dist --project-name=estimate-compare
```

### Staging Deployments (PR Previews)

```yaml
# .github/workflows/preview.yml
name: Preview

on:
  pull_request:
    branches: [main]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
          
      - uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'pnpm'
          
      - run: pnpm install
      
      - name: Deploy Backend to Staging
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        run: |
          cd packages/backend
          npx wrangler deploy --env=staging
          
      - name: Build & Deploy Frontend Preview
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        run: |
          cd packages/frontend
          VITE_API_URL=https://estimate-compare-api-staging.<subdomain>.workers.dev/api pnpm build
          npx wrangler pages deploy dist --project-name=estimate-compare --branch=${{ github.head_ref }}
```

## Monitoring & Logging

### Workers Analytics

Access via Cloudflare Dashboard:
- Workers & Pages > Your Worker > Analytics
- View requests, errors, CPU time

### Custom Logging

```typescript
// packages/backend/src/lib/logger.ts
export function log(level: 'debug' | 'info' | 'warn' | 'error', message: string, data?: unknown) {
  const entry = {
    timestamp: new Date().toISOString(),
    level,
    message,
    ...(data && { data }),
  };
  
  if (level === 'error') {
    console.error(JSON.stringify(entry));
  } else {
    console.log(JSON.stringify(entry));
  }
}

// Usage
log('info', 'Session created', { sessionId, estimateCount: 3 });
log('error', 'Processing failed', { estimateId, error: err.message });
```

### Error Tracking (Optional)

```typescript
// Integration with error tracking service
export function captureError(error: Error, context?: Record<string, unknown>) {
  // Could integrate with Sentry, LogRocket, etc.
  console.error('Error captured:', {
    name: error.name,
    message: error.message,
    stack: error.stack,
    context,
  });
}
```

### Health Checks

```typescript
// packages/backend/src/routes/health.ts
app.get('/api/health', async (c) => {
  const checks: Record<string, boolean> = {};
  
  // Check D1
  try {
    await c.env.DB.prepare('SELECT 1').first();
    checks.database = true;
  } catch {
    checks.database = false;
  }
  
  // Check R2
  try {
    await c.env.R2.head('_health_check');
    checks.storage = true;
  } catch {
    checks.storage = true; // Head on non-existent key is OK
  }
  
  const healthy = Object.values(checks).every(v => v);
  
  return c.json({
    status: healthy ? 'ok' : 'degraded',
    timestamp: new Date().toISOString(),
    checks,
  }, healthy ? 200 : 503);
});
```

## Security Considerations

### CORS Configuration

```typescript
// Restrict in production
app.use('*', cors({
  origin: (origin) => {
    const allowed = [
      'https://estimate-compare.pages.dev',
      'https://app.yourdomain.com',
    ];
    
    if (process.env.ENVIRONMENT === 'development') {
      allowed.push('http://localhost:5173');
    }
    
    return allowed.includes(origin) ? origin : null;
  },
  allowMethods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowHeaders: ['Content-Type'],
}));
```

### Rate Limiting (Future)

```typescript
// Basic rate limiting with KV
async function rateLimit(c: Context, key: string, limit: number, window: number) {
  const count = await c.env.KV.get(`ratelimit:${key}`);
  const current = parseInt(count || '0');
  
  if (current >= limit) {
    return c.json({ error: 'Rate limited' }, 429);
  }
  
  await c.env.KV.put(`ratelimit:${key}`, String(current + 1), {
    expirationTtl: window,
  });
  
  return null;
}
```

### File Upload Validation

```typescript
const MAX_FILE_SIZE = 20 * 1024 * 1024; // 20MB
const MAX_FILES = 10;

// Validate in upload handler
if (files.length > MAX_FILES) {
  throw new ApiError('TOO_MANY_FILES', `Maximum ${MAX_FILES} files allowed`, 400);
}

for (const file of files) {
  if (file.size > MAX_FILE_SIZE) {
    throw new ApiError('FILE_TOO_LARGE', `File ${file.name} exceeds 20MB limit`, 400);
  }
}
```

## Rollback Procedures

### Worker Rollback

```bash
# List deployments
wrangler deployments list

# Rollback to previous deployment
wrangler rollback
```

### Database Rollback

```bash
# Export current data
wrangler d1 export estimate-compare > backup.sql

# If migration fails, restore from backup
wrangler d1 execute estimate-compare --file=backup.sql
```

## Cost Estimation

| Service | Free Tier | Paid Estimate |
|---------|-----------|---------------|
| Workers | 100K req/day | $5/mo + $0.50/M req |
| Pages | Unlimited | Free |
| D1 | 5M rows read/day | $0.001/M rows |
| R2 | 10GB storage | $0.015/GB/mo |
| Workers AI | Limited | Pay per token |

**Estimated monthly cost for moderate usage:** $10-30/month

## Todo List

### Backend Deployment

- [ ] Configure wrangler.toml for production
- [ ] Create production D1 database
- [ ] Create production R2 bucket
- [ ] Run database migrations
- [ ] Deploy worker
- [ ] Verify health endpoint

### Frontend Deployment

- [ ] Configure vite for production
- [ ] Create Pages project
- [ ] Configure build settings
- [ ] Add SPA redirects
- [ ] Deploy to Pages
- [ ] Verify frontend loads

### Domain Setup

- [ ] Configure custom domain for API (optional)
- [ ] Configure custom domain for frontend (optional)
- [ ] Setup SSL (automatic)

### CI/CD

- [ ] Create GitHub Actions workflow
- [ ] Add Cloudflare secrets to GitHub
- [ ] Test deployment pipeline
- [ ] Setup staging environment
- [ ] Configure PR previews

### Monitoring

- [ ] Setup logging
- [ ] Configure health checks
- [ ] Review Workers analytics
- [ ] Setup error alerts (optional)

### Security

- [ ] Restrict CORS origins
- [ ] Validate file uploads
- [ ] Review rate limiting needs
- [ ] Audit API endpoints

## Verification Checklist

- [ ] Backend accessible at production URL
- [ ] Frontend loads and connects to API
- [ ] File upload works in production
- [ ] Database operations succeed
- [ ] R2 storage works
- [ ] AI processing works
- [ ] Custom domain works (if configured)
- [ ] CI/CD pipeline deploys successfully
- [ ] Rollback procedure tested

## Notes

### Workers Limits

- 10ms CPU time (free), 30s (paid)
- 128MB memory
- 100 subrequests per request
- 25MB script size

### D1 Limits

- 10GB max database size
- 256 columns per table
- 1MB max row size

### R2 Limits

- 5TB max object size
- No egress fees

## Time Estimate

| Task | Estimate |
|------|----------|
| Backend deployment | 1 hour |
| Frontend deployment | 45 min |
| Domain setup | 30 min |
| CI/CD setup | 1.5 hours |
| Monitoring | 30 min |
| Security review | 30 min |
| Testing | 1 hour |
| **Total** | **~5.75 hours** |
