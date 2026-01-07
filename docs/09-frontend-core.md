# Module 09: Frontend Core

## Overview

Core frontend setup with Preact, TanStack Query for data fetching, routing, and base application structure.

## Goals

- Setup Preact with TypeScript
- Configure TanStack Query for caching and mutations
- Implement API client
- Create base app layout and routing
- Setup CSS/styling approach

## File Structure

```
packages/frontend/
├── index.html
├── vite.config.ts
├── tsconfig.json
├── package.json
└── src/
    ├── main.tsx              # Entry point
    ├── App.tsx               # Root component with routing
    ├── api/
    │   ├── client.ts         # HTTP client
    │   └── queries.ts        # TanStack Query hooks
    ├── hooks/
    │   ├── usePolling.ts     # Polling utilities
    │   └── useLocalStorage.ts
    ├── lib/
    │   ├── formatting.ts     # Number/currency formatting
    │   └── constants.ts      # App constants
    ├── styles/
    │   ├── main.css          # Global styles
    │   ├── variables.css     # CSS variables
    │   └── components.css    # Component styles
    ├── pages/                # Page components
    └── components/           # Shared components
```

## Core Implementation

### Vite Configuration (vite.config.ts)

```typescript
import { defineConfig } from 'vite';
import preact from '@preact/preset-vite';
import { resolve } from 'path';

export default defineConfig({
  plugins: [preact()],
  
  resolve: {
    alias: {
      // React compatibility for TanStack
      'react': 'preact/compat',
      'react-dom': 'preact/compat',
      'react/jsx-runtime': 'preact/jsx-runtime',
      // Path aliases
      '@': resolve(__dirname, './src'),
    },
  },
  
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8787',
        changeOrigin: true,
      },
    },
  },
  
  build: {
    outDir: 'dist',
    sourcemap: true,
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['preact', '@tanstack/react-query', '@tanstack/react-table'],
        },
      },
    },
  },
});
```

### Entry Point (main.tsx)

```typescript
import { render } from 'preact';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './api/queries';
import { App } from './App';
import './styles/main.css';

render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>,
  document.getElementById('app')!
);
```

### HTML Template (index.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Estimate Compare</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

### Root App Component (App.tsx)

```typescript
import { useState, useEffect } from 'preact/hooks';
import { Upload } from './pages/Upload';
import { Results } from './pages/Results';

export type AppScreen = 
  | { type: 'upload' }
  | { type: 'processing'; sessionId: string }
  | { type: 'results'; sessionId: string };

export function App() {
  const [screen, setScreen] = useState<AppScreen>(() => {
    // Check URL for session ID
    const params = new URLSearchParams(window.location.search);
    const sessionId = params.get('session');
    
    if (sessionId) {
      return { type: 'results', sessionId };
    }
    
    return { type: 'upload' };
  });

  // Update URL when screen changes
  useEffect(() => {
    if (screen.type === 'results' || screen.type === 'processing') {
      const url = new URL(window.location.href);
      url.searchParams.set('session', screen.sessionId);
      window.history.replaceState({}, '', url.toString());
    } else {
      const url = new URL(window.location.href);
      url.searchParams.delete('session');
      window.history.replaceState({}, '', url.toString());
    }
  }, [screen]);

  const handleSessionCreated = (sessionId: string) => {
    setScreen({ type: 'processing', sessionId });
  };

  const handleProcessingComplete = () => {
    if (screen.type === 'processing') {
      setScreen({ type: 'results', sessionId: screen.sessionId });
    }
  };

  const handleNewComparison = () => {
    setScreen({ type: 'upload' });
  };

  return (
    <div class="app">
      <Header 
        showNewButton={screen.type !== 'upload'}
        onNewComparison={handleNewComparison}
      />
      
      <main class="app-main">
        {screen.type === 'upload' && (
          <Upload onSessionCreated={handleSessionCreated} />
        )}
        
        {(screen.type === 'processing' || screen.type === 'results') && (
          <Results
            sessionId={screen.sessionId}
            onReady={handleProcessingComplete}
          />
        )}
      </main>
      
      <Footer />
    </div>
  );
}

function Header({ 
  showNewButton, 
  onNewComparison 
}: { 
  showNewButton: boolean;
  onNewComparison: () => void;
}) {
  return (
    <header class="app-header">
      <div class="header-content">
        <h1 class="header-title">
          <span class="header-icon">📊</span>
          Estimate Compare
        </h1>
        
        {showNewButton && (
          <button 
            class="btn btn-secondary"
            onClick={onNewComparison}
          >
            New Comparison
          </button>
        )}
      </div>
    </header>
  );
}

function Footer() {
  return (
    <footer class="app-footer">
      <p>Construction Estimate Comparison Tool</p>
    </footer>
  );
}
```

### API Client (api/client.ts)

```typescript
const API_BASE = import.meta.env.VITE_API_URL || '/api';

export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

async function handleResponse<T>(response: Response): Promise<T> {
  const contentType = response.headers.get('content-type');
  const isJson = contentType?.includes('application/json');
  
  if (!response.ok) {
    let errorMessage = `Request failed with status ${response.status}`;
    let details: unknown;
    
    if (isJson) {
      try {
        const error = await response.json();
        errorMessage = error.error || error.message || errorMessage;
        details = error.details;
      } catch {
        // Ignore JSON parse errors
      }
    }
    
    throw new ApiError(response.status, errorMessage, details);
  }
  
  if (!isJson) {
    throw new ApiError(500, 'Expected JSON response');
  }
  
  return response.json();
}

export const apiClient = {
  /**
   * GET request
   */
  async get<T>(path: string): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });
    return handleResponse<T>(response);
  },

  /**
   * POST request with JSON body
   */
  async post<T>(path: string, body?: unknown): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
      body: body ? JSON.stringify(body) : undefined,
    });
    return handleResponse<T>(response);
  },

  /**
   * POST request with FormData (for file uploads)
   */
  async postForm<T>(path: string, formData: FormData): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
      method: 'POST',
      // Don't set Content-Type - browser will set it with boundary
      headers: {
        'Accept': 'application/json',
      },
      body: formData,
    });
    return handleResponse<T>(response);
  },

  /**
   * PATCH request
   */
  async patch<T>(path: string, body: unknown): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
      body: JSON.stringify(body),
    });
    return handleResponse<T>(response);
  },

  /**
   * DELETE request
   */
  async delete<T>(path: string): Promise<T> {
    const response = await fetch(`${API_BASE}${path}`, {
      method: 'DELETE',
      headers: {
        'Accept': 'application/json',
      },
    });
    return handleResponse<T>(response);
  },
};
```

### TanStack Query Setup (api/queries.ts)

```typescript
import { 
  QueryClient,
  useQuery, 
  useMutation, 
  useQueryClient,
} from '@tanstack/react-query';
import { apiClient } from './client';
import type {
  Session,
  Estimate,
  Comparison,
  CostCode,
  EstimateHeader,
} from '@estimate-compare/shared';

// ============ QUERY CLIENT ============

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,       // 30 seconds
      gcTime: 5 * 60 * 1000,   // 5 minutes (formerly cacheTime)
      retry: 1,
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: 0,
    },
  },
});

// ============ TYPE DEFINITIONS ============

interface CreateSessionResponse {
  sessionId: string;
  formatId: string;
  status: string;
}

interface UploadResponse {
  sessionId: string;
  estimateCount: number;
  estimates: Array<{ id: string; filename: string }>;
  warnings?: string[];
}

interface SessionStatusResponse {
  id: string;
  status: Session['status'];
  progress: {
    total: number;
    completed: number;
    currentFile?: string;
  };
  ready: boolean;
  errorMessage?: string;
  estimates: EstimateHeader[];
}

interface SessionResponse {
  session: Session;
  estimates: Estimate[];
  comparison: Comparison | null;
}

interface EstimateResponse {
  estimate: Estimate;
}

interface ReclassifyResponse {
  lineItem: LineItem;
  comparison: Comparison;
  sessionId: string;
}

interface CostCodesResponse {
  codes: CostCode[];
}

// ============ QUERY HOOKS ============

/**
 * Create a new session
 */
export function useCreateSession() {
  return useMutation({
    mutationFn: (formatId?: string) =>
      apiClient.post<CreateSessionResponse>('/sessions', { formatId }),
  });
}

/**
 * Upload files to a session
 */
export function useUploadEstimates() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ 
      sessionId, 
      files 
    }: { 
      sessionId: string; 
      files: File[] 
    }) => {
      const formData = new FormData();
      files.forEach(file => formData.append('files', file));
      
      return apiClient.postForm<UploadResponse>(
        `/sessions/${sessionId}/upload`,
        formData
      );
    },
    onSuccess: (_, { sessionId }) => {
      // Invalidate session status to start polling
      queryClient.invalidateQueries({ 
        queryKey: ['session-status', sessionId] 
      });
    },
  });
}

/**
 * Poll session status
 */
export function useSessionStatus(sessionId: string | null) {
  return useQuery({
    queryKey: ['session-status', sessionId],
    queryFn: () => 
      apiClient.get<SessionStatusResponse>(`/sessions/${sessionId}/status`),
    enabled: !!sessionId,
    refetchInterval: (query) => {
      // Stop polling when ready or error
      const status = query.state.data?.status;
      if (status === 'ready' || status === 'error') {
        return false;
      }
      return 1000;  // Poll every second
    },
  });
}

/**
 * Get full session data with comparison
 */
export function useSession(sessionId: string | null) {
  return useQuery({
    queryKey: ['session', sessionId],
    queryFn: () => 
      apiClient.get<SessionResponse>(`/sessions/${sessionId}`),
    enabled: !!sessionId,
  });
}

/**
 * Get single estimate detail
 */
export function useEstimate(estimateId: string | null) {
  return useQuery({
    queryKey: ['estimate', estimateId],
    queryFn: () => 
      apiClient.get<EstimateResponse>(`/estimates/${estimateId}`),
    enabled: !!estimateId,
    select: (data) => data.estimate,
  });
}

/**
 * Reclassify a line item
 */
export function useReclassifyLineItem() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({
      estimateId,
      lineItemId,
      newCode,
    }: {
      estimateId: string;
      lineItemId: string;
      newCode: string;
    }) => {
      return apiClient.patch<ReclassifyResponse>(
        `/estimates/${estimateId}/items/${lineItemId}`,
        { newCode }
      );
    },
    onSuccess: (data, variables) => {
      // Update estimate cache with new line item
      queryClient.setQueryData(
        ['estimate', variables.estimateId],
        (old: Estimate | undefined) => {
          if (!old) return old;
          return {
            ...old,
            lineItems: old.lineItems.map(li =>
              li.id === variables.lineItemId ? data.lineItem : li
            ),
          };
        }
      );

      // Update session cache with new comparison
      queryClient.setQueryData(
        ['session', data.sessionId],
        (old: SessionResponse | undefined) => {
          if (!old) return old;
          return {
            ...old,
            comparison: data.comparison,
          };
        }
      );
    },
  });
}

/**
 * Get cost codes for a format
 */
export function useCostCodes(formatId: string = 'residential-v1') {
  return useQuery({
    queryKey: ['cost-codes', formatId],
    queryFn: () => 
      apiClient.get<CostCodesResponse>(`/cost-codes/${formatId}`),
    staleTime: Infinity,  // Cost codes never change
    select: (data) => data.codes,
  });
}
```

### Formatting Utilities (lib/formatting.ts)

```typescript
/**
 * Format currency value
 */
export function formatCurrency(
  value: number | null | undefined,
  options?: Intl.NumberFormatOptions
): string {
  if (value === null || value === undefined) {
    return '—';
  }

  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0,
    ...options,
  }).format(value);
}

/**
 * Format percentage value
 */
export function formatPercent(
  value: number | null | undefined,
  options?: { decimals?: number }
): string {
  if (value === null || value === undefined) {
    return '—';
  }

  const decimals = options?.decimals ?? 1;
  return `${(value * 100).toFixed(decimals)}%`;
}

/**
 * Format number with thousand separators
 */
export function formatNumber(
  value: number | null | undefined,
  options?: Intl.NumberFormatOptions
): string {
  if (value === null || value === undefined) {
    return '—';
  }

  return new Intl.NumberFormat('en-US', options).format(value);
}

/**
 * Format file size
 */
export function formatFileSize(bytes: number): string {
  if (bytes === 0) return '0 B';
  
  const k = 1024;
  const sizes = ['B', 'KB', 'MB', 'GB'];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  
  return `${parseFloat((bytes / Math.pow(k, i)).toFixed(1))} ${sizes[i]}`;
}

/**
 * Format date/time
 */
export function formatDateTime(
  date: string | Date,
  options?: Intl.DateTimeFormatOptions
): string {
  const d = typeof date === 'string' ? new Date(date) : date;
  
  return new Intl.DateTimeFormat('en-US', {
    dateStyle: 'medium',
    timeStyle: 'short',
    ...options,
  }).format(d);
}
```

### CSS Variables (styles/variables.css)

```css
:root {
  /* Colors */
  --color-primary: #2563eb;
  --color-primary-dark: #1d4ed8;
  --color-primary-light: #3b82f6;
  
  --color-success: #16a34a;
  --color-warning: #ca8a04;
  --color-danger: #dc2626;
  
  --color-gray-50: #f9fafb;
  --color-gray-100: #f3f4f6;
  --color-gray-200: #e5e7eb;
  --color-gray-300: #d1d5db;
  --color-gray-400: #9ca3af;
  --color-gray-500: #6b7280;
  --color-gray-600: #4b5563;
  --color-gray-700: #374151;
  --color-gray-800: #1f2937;
  --color-gray-900: #111827;
  
  --color-background: #ffffff;
  --color-surface: #ffffff;
  --color-text: var(--color-gray-900);
  --color-text-muted: var(--color-gray-500);
  
  /* Typography */
  --font-sans: 'Inter', system-ui, -apple-system, sans-serif;
  --font-mono: ui-monospace, 'Cascadia Code', monospace;
  
  --text-xs: 0.75rem;
  --text-sm: 0.875rem;
  --text-base: 1rem;
  --text-lg: 1.125rem;
  --text-xl: 1.25rem;
  --text-2xl: 1.5rem;
  --text-3xl: 1.875rem;
  
  /* Spacing */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-5: 1.25rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-10: 2.5rem;
  --space-12: 3rem;
  
  /* Border Radius */
  --radius-sm: 0.25rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  
  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
  
  /* Transitions */
  --transition-fast: 150ms ease;
  --transition-normal: 200ms ease;
}
```

### Global Styles (styles/main.css)

```css
@import './variables.css';
@import './components.css';

/* Reset */
*, *::before, *::after {
  box-sizing: border-box;
}

* {
  margin: 0;
}

html {
  font-size: 16px;
  -webkit-font-smoothing: antialiased;
}

body {
  font-family: var(--font-sans);
  color: var(--color-text);
  background: var(--color-gray-50);
  line-height: 1.5;
  min-height: 100vh;
}

/* App Layout */
.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
}

.app-header {
  background: var(--color-background);
  border-bottom: 1px solid var(--color-gray-200);
  padding: var(--space-4) var(--space-6);
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-content {
  max-width: 1400px;
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.header-title {
  font-size: var(--text-xl);
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: var(--space-2);
}

.header-icon {
  font-size: var(--text-2xl);
}

.app-main {
  flex: 1;
  padding: var(--space-6);
  max-width: 1400px;
  margin: 0 auto;
  width: 100%;
}

.app-footer {
  background: var(--color-background);
  border-top: 1px solid var(--color-gray-200);
  padding: var(--space-4) var(--space-6);
  text-align: center;
  color: var(--color-text-muted);
  font-size: var(--text-sm);
}

/* Utility Classes */
.text-muted {
  color: var(--color-text-muted);
}

.text-success {
  color: var(--color-success);
}

.text-warning {
  color: var(--color-warning);
}

.text-danger {
  color: var(--color-danger);
}

.font-bold {
  font-weight: 600;
}

.font-mono {
  font-family: var(--font-mono);
}

/* Loading States */
.loading {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: var(--space-12);
  color: var(--color-text-muted);
}

.loading-spinner {
  width: 24px;
  height: 24px;
  border: 2px solid var(--color-gray-200);
  border-top-color: var(--color-primary);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin-right: var(--space-3);
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

/* Error States */
.error-message {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: var(--color-danger);
  padding: var(--space-3) var(--space-4);
  border-radius: var(--radius-md);
  font-size: var(--text-sm);
}
```

### Component Styles (styles/components.css)

```css
/* Buttons */
.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: var(--space-2);
  padding: var(--space-2) var(--space-4);
  font-size: var(--text-sm);
  font-weight: 500;
  border-radius: var(--radius-md);
  border: 1px solid transparent;
  cursor: pointer;
  transition: all var(--transition-fast);
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background: var(--color-primary);
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: var(--color-primary-dark);
}

.btn-secondary {
  background: var(--color-background);
  border-color: var(--color-gray-300);
  color: var(--color-gray-700);
}

.btn-secondary:hover:not(:disabled) {
  background: var(--color-gray-50);
  border-color: var(--color-gray-400);
}

.btn-large {
  padding: var(--space-3) var(--space-6);
  font-size: var(--text-base);
}

/* Cards */
.card {
  background: var(--color-background);
  border: 1px solid var(--color-gray-200);
  border-radius: var(--radius-lg);
  padding: var(--space-6);
  box-shadow: var(--shadow-sm);
}

/* Tables */
table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--text-sm);
}

th, td {
  padding: var(--space-3) var(--space-4);
  text-align: left;
  border-bottom: 1px solid var(--color-gray-200);
}

th {
  font-weight: 600;
  background: var(--color-gray-50);
  position: sticky;
  top: 0;
}

tr:hover td {
  background: var(--color-gray-50);
}

/* Forms */
.input {
  width: 100%;
  padding: var(--space-2) var(--space-3);
  font-size: var(--text-sm);
  border: 1px solid var(--color-gray-300);
  border-radius: var(--radius-md);
  transition: border-color var(--transition-fast);
}

.input:focus {
  outline: none;
  border-color: var(--color-primary);
  box-shadow: 0 0 0 3px rgb(37 99 235 / 0.1);
}

/* Badges */
.badge {
  display: inline-flex;
  align-items: center;
  padding: var(--space-1) var(--space-2);
  font-size: var(--text-xs);
  font-weight: 500;
  border-radius: var(--radius-sm);
}

.badge-success {
  background: #dcfce7;
  color: var(--color-success);
}

.badge-warning {
  background: #fef3c7;
  color: var(--color-warning);
}

.badge-danger {
  background: #fef2f2;
  color: var(--color-danger);
}

.badge-gray {
  background: var(--color-gray-100);
  color: var(--color-gray-600);
}

/* Modal */
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgb(0 0 0 / 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: var(--space-4);
}

.modal {
  background: var(--color-background);
  border-radius: var(--radius-xl);
  box-shadow: var(--shadow-lg);
  max-width: 500px;
  width: 100%;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-4) var(--space-6);
  border-bottom: 1px solid var(--color-gray-200);
}

.modal-header h3 {
  font-size: var(--text-lg);
  font-weight: 600;
}

.modal-close {
  background: none;
  border: none;
  font-size: var(--text-xl);
  color: var(--color-gray-400);
  cursor: pointer;
  padding: var(--space-1);
}

.modal-close:hover {
  color: var(--color-gray-600);
}

.modal-body {
  padding: var(--space-6);
  overflow-y: auto;
  flex: 1;
}

.modal-footer {
  display: flex;
  justify-content: flex-end;
  gap: var(--space-3);
  padding: var(--space-4) var(--space-6);
  border-top: 1px solid var(--color-gray-200);
}

/* Tabs */
.tabs {
  display: flex;
  border-bottom: 1px solid var(--color-gray-200);
  margin-bottom: var(--space-6);
  overflow-x: auto;
}

.tab {
  padding: var(--space-3) var(--space-4);
  font-size: var(--text-sm);
  font-weight: 500;
  color: var(--color-text-muted);
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  cursor: pointer;
  white-space: nowrap;
  transition: all var(--transition-fast);
}

.tab:hover {
  color: var(--color-text);
}

.tab.active {
  color: var(--color-primary);
  border-bottom-color: var(--color-primary);
}
```

## Todo List

### Project Setup

- [ ] Create package.json with dependencies
- [ ] Create vite.config.ts with aliases and proxy
- [ ] Create tsconfig.json
- [ ] Create index.html template
- [ ] Verify Preact/React compatibility

### Core Files

- [ ] Create src/main.tsx entry point
- [ ] Create src/App.tsx with routing
- [ ] Implement Header component
- [ ] Implement Footer component
- [ ] Add URL state management

### API Layer

- [ ] Create api/client.ts
- [ ] Implement ApiError class
- [ ] Implement handleResponse helper
- [ ] Implement get, post, postForm, patch methods
- [ ] Create api/queries.ts
- [ ] Configure QueryClient
- [ ] Implement useCreateSession hook
- [ ] Implement useUploadEstimates hook
- [ ] Implement useSessionStatus hook (with polling)
- [ ] Implement useSession hook
- [ ] Implement useEstimate hook
- [ ] Implement useReclassifyLineItem hook
- [ ] Implement useCostCodes hook

### Utilities

- [ ] Create lib/formatting.ts
- [ ] Implement formatCurrency
- [ ] Implement formatPercent
- [ ] Implement formatNumber
- [ ] Implement formatFileSize
- [ ] Implement formatDateTime

### Styles

- [ ] Create styles/variables.css
- [ ] Create styles/main.css
- [ ] Create styles/components.css
- [ ] Style buttons
- [ ] Style cards
- [ ] Style tables
- [ ] Style forms/inputs
- [ ] Style badges
- [ ] Style modals
- [ ] Style tabs

### Testing

- [ ] Verify app renders
- [ ] Verify API client works
- [ ] Verify TanStack Query caching
- [ ] Verify polling behavior
- [ ] Verify styles render correctly
- [ ] Test responsive layout

## Verification Checklist

- [ ] App renders without errors
- [ ] TanStack Query provider configured
- [ ] API calls work through proxy
- [ ] Polling stops when ready/error
- [ ] URL reflects current session
- [ ] Styles are consistent
- [ ] TypeScript has no errors

## Notes

### Preact + TanStack Compatibility

The key is aliasing React to Preact in vite.config.ts:

```typescript
resolve: {
  alias: {
    'react': 'preact/compat',
    'react-dom': 'preact/compat',
  },
}
```

### Query Key Conventions

```typescript
// Session status (for polling)
['session-status', sessionId]

// Full session data
['session', sessionId]

// Single estimate
['estimate', estimateId]

// Cost codes (static)
['cost-codes', formatId]
```

### CSS Approach

Using plain CSS with:
- CSS custom properties (variables)
- BEM-like class naming
- Minimal utility classes
- Component-specific styles

## Time Estimate

| Task | Estimate |
|------|----------|
| Project setup | 30 min |
| Core files (App, main) | 45 min |
| API client | 45 min |
| Query hooks | 1.5 hours |
| Utilities | 30 min |
| Styles | 1.5 hours |
| Testing | 45 min |
| **Total** | **~6.25 hours** |
