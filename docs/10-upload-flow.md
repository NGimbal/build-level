# Module 10: Upload Flow

## Overview

The upload page with drag-and-drop file selection, file validation, upload progress, and processing status display.

## Goals

- Implement drag-and-drop file upload
- Validate file types and sizes
- Show upload and processing progress
- Handle errors gracefully
- Provide clear feedback to users

## File Structure

```
packages/frontend/src/
├── pages/
│   └── Upload.tsx            # Upload page
├── components/
│   ├── FileDropzone.tsx      # Drag-drop component
│   ├── FileList.tsx          # Selected files list
│   └── ProcessingStatus.tsx  # Processing indicator
```

## Core Implementation

### Upload Page (pages/Upload.tsx)

```typescript
import { useState, useCallback } from 'preact/hooks';
import { useCreateSession, useUploadEstimates } from '../api/queries';
import { FileDropzone } from '../components/FileDropzone';
import { FileList } from '../components/FileList';

interface UploadProps {
  onSessionCreated: (sessionId: string) => void;
}

export function Upload({ onSessionCreated }: UploadProps) {
  const [files, setFiles] = useState<File[]>([]);
  const [error, setError] = useState<string | null>(null);

  const createSession = useCreateSession();
  const uploadEstimates = useUploadEstimates();

  const isUploading = createSession.isPending || uploadEstimates.isPending;

  // Handle file selection from dropzone
  const handleFilesSelected = useCallback((newFiles: File[]) => {
    setError(null);

    const validFiles: File[] = [];
    const errors: string[] = [];

    for (const file of newFiles) {
      // Check file type
      if (!file.name.toLowerCase().endsWith('.pdf')) {
        errors.push(`"${file.name}" is not a PDF file`);
        continue;
      }

      // Check file size (max 20MB)
      if (file.size > 20 * 1024 * 1024) {
        errors.push(`"${file.name}" exceeds 20MB limit`);
        continue;
      }

      // Check for duplicates
      if (files.some(f => f.name === file.name && f.size === file.size)) {
        errors.push(`"${file.name}" is already added`);
        continue;
      }

      validFiles.push(file);
    }

    if (errors.length > 0) {
      setError(errors.join('. '));
    }

    if (validFiles.length > 0) {
      setFiles(prev => [...prev, ...validFiles]);
    }
  }, [files]);

  // Remove a file from the list
  const handleRemoveFile = useCallback((index: number) => {
    setFiles(prev => prev.filter((_, i) => i !== index));
    setError(null);
  }, []);

  // Clear all files
  const handleClearFiles = useCallback(() => {
    setFiles([]);
    setError(null);
  }, []);

  // Start the upload and processing
  const handleSubmit = async () => {
    if (files.length < 2) {
      setError('Please add at least 2 estimates to compare');
      return;
    }

    try {
      setError(null);

      // Create session
      const { sessionId } = await createSession.mutateAsync();

      // Upload files
      const result = await uploadEstimates.mutateAsync({
        sessionId,
        files,
      });

      // Show warnings if any
      if (result.warnings && result.warnings.length > 0) {
        console.warn('Upload warnings:', result.warnings);
      }

      // Navigate to processing/results
      onSessionCreated(sessionId);

    } catch (err) {
      console.error('Upload error:', err);
      setError(
        err instanceof Error 
          ? err.message 
          : 'Upload failed. Please try again.'
      );
    }
  };

  return (
    <div class="upload-page">
      <div class="upload-container card">
        <div class="upload-header">
          <h2>Compare Construction Estimates</h2>
          <p class="upload-subtitle">
            Upload 2 or more PDF estimates to compare them side by side.
            We'll extract line items and categorize them automatically.
          </p>
        </div>

        <FileDropzone
          onFilesSelected={handleFilesSelected}
          disabled={isUploading}
        />

        {files.length > 0 && (
          <FileList
            files={files}
            onRemove={handleRemoveFile}
            onClear={handleClearFiles}
            disabled={isUploading}
          />
        )}

        {error && (
          <div class="error-message">
            {error}
          </div>
        )}

        <div class="upload-actions">
          <button
            class="btn btn-primary btn-large"
            onClick={handleSubmit}
            disabled={files.length < 2 || isUploading}
          >
            {isUploading ? (
              <>
                <span class="loading-spinner"></span>
                Uploading...
              </>
            ) : (
              `Compare ${files.length} Estimate${files.length !== 1 ? 's' : ''}`
            )}
          </button>

          {files.length < 2 && files.length > 0 && (
            <p class="upload-hint">
              Add {2 - files.length} more estimate{2 - files.length > 1 ? 's' : ''} to compare
            </p>
          )}
        </div>
      </div>

      <div class="upload-features">
        <div class="feature">
          <span class="feature-icon">📄</span>
          <h3>PDF Only</h3>
          <p>Upload construction estimates in PDF format</p>
        </div>
        <div class="feature">
          <span class="feature-icon">🤖</span>
          <h3>AI-Powered</h3>
          <p>Line items are extracted and categorized automatically</p>
        </div>
        <div class="feature">
          <span class="feature-icon">📊</span>
          <h3>Side-by-Side</h3>
          <p>Compare costs across estimates by category</p>
        </div>
      </div>
    </div>
  );
}
```

### File Dropzone Component (components/FileDropzone.tsx)

```typescript
import { useState, useRef, useCallback } from 'preact/hooks';

interface FileDropzoneProps {
  onFilesSelected: (files: File[]) => void;
  disabled?: boolean;
  accept?: string;
}

export function FileDropzone({
  onFilesSelected,
  disabled = false,
  accept = '.pdf',
}: FileDropzoneProps) {
  const [isDragActive, setIsDragActive] = useState(false);
  const inputRef = useRef<HTMLInputElement>(null);

  const handleDragEnter = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    if (!disabled) {
      setIsDragActive(true);
    }
  }, [disabled]);

  const handleDragLeave = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragActive(false);
  }, []);

  const handleDragOver = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
  }, []);

  const handleDrop = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragActive(false);

    if (disabled) return;

    const files = e.dataTransfer?.files;
    if (files && files.length > 0) {
      onFilesSelected(Array.from(files));
    }
  }, [disabled, onFilesSelected]);

  const handleInputChange = useCallback((e: Event) => {
    const target = e.target as HTMLInputElement;
    const files = target.files;
    if (files && files.length > 0) {
      onFilesSelected(Array.from(files));
      // Reset input so same file can be selected again
      target.value = '';
    }
  }, [onFilesSelected]);

  const handleClick = useCallback(() => {
    if (!disabled && inputRef.current) {
      inputRef.current.click();
    }
  }, [disabled]);

  const handleKeyDown = useCallback((e: KeyboardEvent) => {
    if ((e.key === 'Enter' || e.key === ' ') && !disabled) {
      e.preventDefault();
      inputRef.current?.click();
    }
  }, [disabled]);

  return (
    <div
      class={`dropzone ${isDragActive ? 'dropzone-active' : ''} ${disabled ? 'dropzone-disabled' : ''}`}
      onDragEnter={handleDragEnter}
      onDragLeave={handleDragLeave}
      onDragOver={handleDragOver}
      onDrop={handleDrop}
      onClick={handleClick}
      onKeyDown={handleKeyDown}
      tabIndex={disabled ? -1 : 0}
      role="button"
      aria-label="Upload PDF files"
    >
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        multiple
        onChange={handleInputChange}
        class="dropzone-input"
        disabled={disabled}
      />

      <div class="dropzone-content">
        <div class="dropzone-icon">
          {isDragActive ? '📂' : '📁'}
        </div>
        <p class="dropzone-text">
          {isDragActive
            ? 'Drop files here'
            : 'Drag & drop PDF files here, or click to browse'}
        </p>
        <p class="dropzone-hint">
          Maximum file size: 20MB
        </p>
      </div>
    </div>
  );
}
```

### File List Component (components/FileList.tsx)

```typescript
import { formatFileSize } from '../lib/formatting';

interface FileListProps {
  files: File[];
  onRemove: (index: number) => void;
  onClear: () => void;
  disabled?: boolean;
}

export function FileList({
  files,
  onRemove,
  onClear,
  disabled = false,
}: FileListProps) {
  const totalSize = files.reduce((sum, f) => sum + f.size, 0);

  return (
    <div class="file-list">
      <div class="file-list-header">
        <h3>Selected Files ({files.length})</h3>
        <div class="file-list-actions">
          <span class="file-list-size">
            Total: {formatFileSize(totalSize)}
          </span>
          <button
            class="btn btn-secondary btn-sm"
            onClick={onClear}
            disabled={disabled}
          >
            Clear All
          </button>
        </div>
      </div>

      <ul class="file-list-items">
        {files.map((file, index) => (
          <li key={`${file.name}-${file.size}-${index}`} class="file-item">
            <div class="file-item-icon">📄</div>
            <div class="file-item-info">
              <span class="file-item-name">{file.name}</span>
              <span class="file-item-size">{formatFileSize(file.size)}</span>
            </div>
            <button
              class="file-item-remove"
              onClick={() => onRemove(index)}
              disabled={disabled}
              aria-label={`Remove ${file.name}`}
            >
              ×
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### Processing Status Component (components/ProcessingStatus.tsx)

```typescript
import type { EstimateHeader } from '@estimate-compare/shared';

interface ProcessingStatusProps {
  status: string;
  progress?: {
    total: number;
    completed: number;
    currentFile?: string;
  };
  estimates: EstimateHeader[];
  errorMessage?: string;
}

export function ProcessingStatus({
  status,
  progress,
  estimates,
  errorMessage,
}: ProcessingStatusProps) {
  const getStatusIcon = (estStatus: string) => {
    switch (estStatus) {
      case 'ready':
        return '✓';
      case 'error':
        return '✗';
      case 'extracting':
      case 'classifying':
        return '⏳';
      default:
        return '○';
    }
  };

  const getStatusLabel = (estStatus: string) => {
    switch (estStatus) {
      case 'pending':
        return 'Waiting...';
      case 'extracting':
        return 'Extracting text...';
      case 'classifying':
        return 'Classifying items...';
      case 'ready':
        return 'Complete';
      case 'error':
        return 'Failed';
      default:
        return estStatus;
    }
  };

  const progressPercent = progress 
    ? Math.round((progress.completed / progress.total) * 100)
    : 0;

  return (
    <div class="processing-status card">
      <div class="processing-header">
        <h2>
          {status === 'error' ? 'Processing Failed' : 'Processing Estimates'}
        </h2>
        {status !== 'error' && progress && (
          <span class="processing-percent">{progressPercent}%</span>
        )}
      </div>

      {status === 'error' && errorMessage && (
        <div class="error-message">
          {errorMessage}
        </div>
      )}

      {status !== 'error' && (
        <>
          <div class="progress-bar">
            <div 
              class="progress-bar-fill"
              style={{ width: `${progressPercent}%` }}
            />
          </div>

          {progress?.currentFile && (
            <p class="processing-current">
              Currently processing: {progress.currentFile}
            </p>
          )}
        </>
      )}

      <div class="processing-estimates">
        <h3>Estimates</h3>
        <ul class="estimate-status-list">
          {estimates.map((est) => (
            <li 
              key={est.id} 
              class={`estimate-status-item status-${est.status}`}
            >
              <span class="estimate-status-icon">
                {getStatusIcon(est.status)}
              </span>
              <span class="estimate-status-name">{est.filename}</span>
              <span class="estimate-status-label">
                {getStatusLabel(est.status)}
              </span>
            </li>
          ))}
        </ul>
      </div>

      {status !== 'error' && (
        <p class="processing-hint">
          This may take a minute. You can leave this page and come back.
        </p>
      )}
    </div>
  );
}
```

### Upload Page Styles

```css
/* Add to styles/components.css */

/* Upload Page */
.upload-page {
  max-width: 800px;
  margin: 0 auto;
}

.upload-container {
  margin-bottom: var(--space-8);
}

.upload-header {
  text-align: center;
  margin-bottom: var(--space-6);
}

.upload-header h2 {
  font-size: var(--text-2xl);
  font-weight: 600;
  margin-bottom: var(--space-2);
}

.upload-subtitle {
  color: var(--color-text-muted);
}

.upload-actions {
  text-align: center;
  margin-top: var(--space-6);
}

.upload-hint {
  margin-top: var(--space-3);
  color: var(--color-text-muted);
  font-size: var(--text-sm);
}

.upload-features {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: var(--space-6);
}

.feature {
  text-align: center;
}

.feature-icon {
  font-size: 2rem;
  display: block;
  margin-bottom: var(--space-2);
}

.feature h3 {
  font-size: var(--text-base);
  font-weight: 600;
  margin-bottom: var(--space-1);
}

.feature p {
  font-size: var(--text-sm);
  color: var(--color-text-muted);
}

/* Dropzone */
.dropzone {
  border: 2px dashed var(--color-gray-300);
  border-radius: var(--radius-lg);
  padding: var(--space-10) var(--space-6);
  text-align: center;
  cursor: pointer;
  transition: all var(--transition-normal);
  background: var(--color-gray-50);
}

.dropzone:hover,
.dropzone:focus {
  border-color: var(--color-primary);
  background: rgba(37, 99, 235, 0.05);
  outline: none;
}

.dropzone-active {
  border-color: var(--color-primary);
  background: rgba(37, 99, 235, 0.1);
  border-style: solid;
}

.dropzone-disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.dropzone-disabled:hover {
  border-color: var(--color-gray-300);
  background: var(--color-gray-50);
}

.dropzone-input {
  display: none;
}

.dropzone-icon {
  font-size: 3rem;
  margin-bottom: var(--space-3);
}

.dropzone-text {
  font-size: var(--text-base);
  font-weight: 500;
  margin-bottom: var(--space-2);
}

.dropzone-hint {
  font-size: var(--text-sm);
  color: var(--color-text-muted);
}

/* File List */
.file-list {
  margin-top: var(--space-6);
  border: 1px solid var(--color-gray-200);
  border-radius: var(--radius-lg);
  overflow: hidden;
}

.file-list-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: var(--space-3) var(--space-4);
  background: var(--color-gray-50);
  border-bottom: 1px solid var(--color-gray-200);
}

.file-list-header h3 {
  font-size: var(--text-sm);
  font-weight: 600;
}

.file-list-actions {
  display: flex;
  align-items: center;
  gap: var(--space-4);
}

.file-list-size {
  font-size: var(--text-sm);
  color: var(--color-text-muted);
}

.btn-sm {
  padding: var(--space-1) var(--space-2);
  font-size: var(--text-xs);
}

.file-list-items {
  list-style: none;
  padding: 0;
  margin: 0;
}

.file-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-3) var(--space-4);
  border-bottom: 1px solid var(--color-gray-100);
}

.file-item:last-child {
  border-bottom: none;
}

.file-item-icon {
  font-size: var(--text-lg);
}

.file-item-info {
  flex: 1;
  min-width: 0;
}

.file-item-name {
  display: block;
  font-size: var(--text-sm);
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-item-size {
  font-size: var(--text-xs);
  color: var(--color-text-muted);
}

.file-item-remove {
  background: none;
  border: none;
  font-size: var(--text-xl);
  color: var(--color-gray-400);
  cursor: pointer;
  padding: var(--space-1);
  line-height: 1;
}

.file-item-remove:hover {
  color: var(--color-danger);
}

.file-item-remove:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Processing Status */
.processing-status {
  max-width: 600px;
  margin: 0 auto;
}

.processing-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--space-4);
}

.processing-header h2 {
  font-size: var(--text-xl);
  font-weight: 600;
}

.processing-percent {
  font-size: var(--text-2xl);
  font-weight: 600;
  color: var(--color-primary);
}

.progress-bar {
  height: 8px;
  background: var(--color-gray-200);
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: var(--space-4);
}

.progress-bar-fill {
  height: 100%;
  background: var(--color-primary);
  border-radius: 4px;
  transition: width var(--transition-normal);
}

.processing-current {
  font-size: var(--text-sm);
  color: var(--color-text-muted);
  margin-bottom: var(--space-6);
}

.processing-estimates h3 {
  font-size: var(--text-sm);
  font-weight: 600;
  margin-bottom: var(--space-3);
  color: var(--color-text-muted);
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.estimate-status-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.estimate-status-item {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: var(--space-2) 0;
  font-size: var(--text-sm);
}

.estimate-status-icon {
  width: 20px;
  height: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  font-size: var(--text-xs);
}

.status-pending .estimate-status-icon {
  background: var(--color-gray-200);
  color: var(--color-gray-500);
}

.status-extracting .estimate-status-icon,
.status-classifying .estimate-status-icon {
  background: #dbeafe;
  color: var(--color-primary);
}

.status-ready .estimate-status-icon {
  background: #dcfce7;
  color: var(--color-success);
}

.status-error .estimate-status-icon {
  background: #fef2f2;
  color: var(--color-danger);
}

.estimate-status-name {
  flex: 1;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.estimate-status-label {
  color: var(--color-text-muted);
}

.processing-hint {
  margin-top: var(--space-6);
  padding-top: var(--space-4);
  border-top: 1px solid var(--color-gray-200);
  font-size: var(--text-sm);
  color: var(--color-text-muted);
  text-align: center;
}
```

## Todo List

### Upload Page

- [ ] Create pages/Upload.tsx
- [ ] Implement file state management
- [ ] Implement handleFilesSelected with validation
- [ ] Implement handleRemoveFile
- [ ] Implement handleClearFiles
- [ ] Implement handleSubmit (create session + upload)
- [ ] Add error display
- [ ] Add loading state
- [ ] Add feature highlights section

### File Dropzone

- [ ] Create components/FileDropzone.tsx
- [ ] Implement drag events (enter, leave, over, drop)
- [ ] Implement file input with ref
- [ ] Implement click-to-browse
- [ ] Implement keyboard accessibility
- [ ] Add active state styling
- [ ] Add disabled state

### File List

- [ ] Create components/FileList.tsx
- [ ] Display file list with icons
- [ ] Show file sizes
- [ ] Implement remove button
- [ ] Implement clear all button
- [ ] Calculate and display total size

### Processing Status

- [ ] Create components/ProcessingStatus.tsx
- [ ] Display overall progress bar
- [ ] Display current file being processed
- [ ] List all estimates with status icons
- [ ] Handle error state display
- [ ] Add informational message

### Styles

- [ ] Add upload page styles
- [ ] Add dropzone styles with states
- [ ] Add file list styles
- [ ] Add processing status styles
- [ ] Add progress bar animation
- [ ] Ensure responsive layout

### Testing

- [ ] Test drag and drop
- [ ] Test click to browse
- [ ] Test file validation (type, size)
- [ ] Test duplicate detection
- [ ] Test file removal
- [ ] Test upload flow
- [ ] Test error handling
- [ ] Test keyboard navigation

## Verification Checklist

- [ ] Drag and drop works
- [ ] Click to browse works
- [ ] Invalid files are rejected with message
- [ ] File list updates correctly
- [ ] Remove buttons work
- [ ] Clear all works
- [ ] Submit button disabled with < 2 files
- [ ] Upload shows loading state
- [ ] Processing status shows progress
- [ ] Errors are displayed clearly

## Notes

### File Validation Rules

| Rule | Limit | Error Message |
|------|-------|---------------|
| File type | .pdf only | "is not a PDF file" |
| File size | 20MB max | "exceeds 20MB limit" |
| Duplicates | Prevented | "is already added" |

### Processing States

```
pending → extracting → classifying → ready
                                  ↘ error
```

### Accessibility

- Dropzone is focusable and keyboard accessible
- Remove buttons have aria-labels
- Progress is announced to screen readers
- Color is not the only indicator of status

## Time Estimate

| Task | Estimate |
|------|----------|
| Upload page | 1.5 hours |
| File dropzone | 1 hour |
| File list | 45 min |
| Processing status | 1 hour |
| Styles | 1.5 hours |
| Testing | 1 hour |
| **Total** | **~6.75 hours** |
