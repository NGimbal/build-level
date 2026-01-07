# Module 01: Project Setup

## Overview

Initial project scaffolding including monorepo configuration, TypeScript setup, and Cloudflare resource provisioning.

## Goals

- Create a pnpm monorepo with three packages (shared, backend, frontend)
- Configure TypeScript with strict mode
- Setup Cloudflare Workers with Hono
- Provision D1 database and R2 bucket
- Configure development workflow

## File Structure

```
build-level/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── .gitignore
├── .nvmrc
├── packages/
│   ├── shared/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       └── index.ts
│   ├── backend/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── wrangler.toml
│   │   └── src/
│   │       └── index.ts
│   └── frontend/
│       ├── package.json
│       ├── tsconfig.json
│       ├── vite.config.ts
│       ├── index.html
│       └── src/
│           └── main.tsx
```

## Configuration Files

### Root package.json

```json
{
  "name": "build-level",
  "private": true,
  "scripts": {
    "dev": "pnpm --parallel --filter './packages/*' dev",
    "build": "pnpm --filter './packages/*' build",
    "typecheck": "pnpm --filter './packages/*' typecheck",
    "clean": "pnpm --filter './packages/*' clean"
  },
  "devDependencies": {
    "typescript": "^5.3.3"
  }
}
```

### pnpm-workspace.yaml

```yaml
packages:
  - "packages/*"
```

### tsconfig.base.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

### Backend wrangler.toml

```toml
name = "build-level-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[[d1_databases]]
binding = "DB"
database_name = "build-level"
database_id = "<your-database-id>"

[[r2_buckets]]
binding = "R2"
bucket_name = "build-level-files"

[ai]
binding = "AI"

[env.dev]
name = "build-level-api-dev"
```

### Frontend vite.config.ts

```typescript
import { defineConfig } from "vite";
import preact from "@preact/preset-vite";

export default defineConfig({
  plugins: [preact()],
  server: {
    port: 5173,
    proxy: {
      "/api": {
        target: "http://localhost:8787",
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: "dist",
    sourcemap: true,
  },
});
```

## Dependencies

### Backend (packages/backend/package.json)

```json
{
  "name": "@build-level/backend",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "wrangler dev",
    "build": "wrangler deploy --dry-run",
    "deploy": "wrangler deploy",
    "typecheck": "tsc --noEmit",
    "db:migrate": "wrangler d1 execute build-level --file=./src/db/schema.sql",
    "db:seed": "wrangler d1 execute build-level --file=./src/db/seed.sql"
  },
  "dependencies": {
    "hono": "^4.0.0",
    "pdfjs-serverless": "^0.4.0",
    "@build-level/shared": "workspace:*"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20240117.0",
    "wrangler": "^3.24.0",
    "typescript": "^5.3.3"
  }
}
```

### Frontend (packages/frontend/package.json)

```json
{
  "name": "@build-level/frontend",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "preact": "^10.19.3",
    "@tanstack/react-query": "^5.17.0",
    "@tanstack/react-table": "^8.11.0",
    "@build-level/shared": "workspace:*"
  },
  "devDependencies": {
    "@preact/preset-vite": "^2.8.1",
    "vite": "^5.0.11",
    "typescript": "^5.3.3"
  }
}
```

## Cloudflare Setup Commands

```bash
# Login to Cloudflare
wrangler login

# Create D1 database
wrangler d1 create build-level
# Note the database ID from output and add to wrangler.toml

# Create R2 bucket
wrangler r2 bucket create build-level-files

# Verify AI is available
wrangler ai models

# Test local development
cd packages/backend
wrangler dev
```

## Todo List

### Project Initialization

- [ ] Create root directory structure
- [ ] Initialize pnpm workspace
- [ ] Create root package.json with workspace scripts
- [ ] Create pnpm-workspace.yaml
- [ ] Create tsconfig.base.json
- [ ] Create .gitignore
- [ ] Create .nvmrc (node version)

### Shared Package

- [ ] Create packages/shared directory
- [ ] Initialize package.json with dependencies
- [ ] Create tsconfig.json extending base
- [ ] Create src/index.ts with placeholder export

### Backend Package

- [ ] Create packages/backend directory
- [ ] Initialize package.json with dependencies
- [ ] Create tsconfig.json extending base
- [ ] Create wrangler.toml template
- [ ] Create src/index.ts with basic Hono app
- [ ] Verify `wrangler dev` starts successfully

### Frontend Package

- [ ] Create packages/frontend directory
- [ ] Initialize package.json with dependencies
- [ ] Create tsconfig.json extending base
- [ ] Create vite.config.ts with proxy
- [ ] Create index.html
- [ ] Create src/main.tsx with minimal Preact app
- [ ] Verify `pnpm dev` starts successfully

### Cloudflare Resources

- [ ] Login to Cloudflare via wrangler
- [ ] Create D1 database
- [ ] Update wrangler.toml with database ID
- [ ] Create R2 bucket
- [ ] Verify Workers AI access
- [ ] Test local development with all bindings

### Development Workflow

- [ ] Verify `pnpm dev` runs all packages
- [ ] Verify frontend proxies to backend
- [ ] Verify TypeScript compilation
- [ ] Create initial git commit

## Verification Checklist

- [ ] `pnpm install` completes without errors
- [ ] `pnpm dev` starts both frontend and backend
- [ ] Frontend accessible at http://localhost:5173
- [ ] Backend accessible at http://localhost:8787
- [ ] API proxy works (frontend can call /api/health)
- [ ] TypeScript shows no errors
- [ ] D1 database is accessible locally
- [ ] R2 bucket is accessible locally
- [ ] Workers AI is available

## Notes

### Known Issues

1. **pdfjs-serverless compatibility**: May need `nodejs_compat` flag in wrangler.toml
2. **TanStack packages**: Use React aliases in Preact for compatibility
3. **D1 local development**: Uses local SQLite file, not remote database

### Preact + TanStack Compatibility

Add to vite.config.ts for proper React aliasing:

```typescript
resolve: {
  alias: {
    'react': 'preact/compat',
    'react-dom': 'preact/compat',
    'react/jsx-runtime': 'preact/jsx-runtime',
  },
},
```

## Time Estimate

| Task                         | Estimate       |
| ---------------------------- | -------------- |
| Project initialization       | 30 min         |
| Package scaffolding          | 45 min         |
| Cloudflare resource setup    | 30 min         |
| Configuration & verification | 45 min         |
| **Total**                    | **~2.5 hours** |
