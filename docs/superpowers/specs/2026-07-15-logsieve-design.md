# LogSieve Design Specification

LogSieve is a local-first, high-performance server log explorer and analyzer that runs entirely in the browser. It features a dual-panel layout, visual metrics graphs, click-to-filter categories, and SQL-powered searching using an in-memory SQLite WebAssembly database inside a Web Worker.

## Goal Description
Developers and QA engineers often need to inspect, query, and analyze massive server logs (e.g., Nginx, Apache, Spring Boot) during local debugging or post-mortem incident reviews. Existing online tools are either slow, insecure (requiring uploading sensitive logs to external servers), or require writing custom scripting code. LogSieve solves this by providing a local-first, fast, and visually premium interface that parses and indexes log files inside the browser using WebAssembly.

## Design Highlights
*   **Dual-Panel Command Layout**: Left side contains visual health metrics (error rates, response times) and quick-filters, while the right side displays a monospace log stream console.
*   **SQL-Powered Database Engine**: Raw log lines are parsed into tables using a `sql.js` (SQLite Wasm) engine running inside a background Web Worker.
*   **Zero-External Costs & Total Privacy**: Files never leave the browser; all computations run client-side.
*   **Virtual List Rendering**: Log stream viewer handles 100k+ rows easily by rendering only the visible viewport.

## User Review Required
> [!NOTE]
> All file parsing, SQLite indexing, and SQL queries run in the browser using WebAssembly. There are no server-side API or database components.

## Proposed System Changes

### 1. File Structure
The project will be created in a new workspace directory at `D:\logsieve` using Vite, React, TypeScript, and Tailwind CSS.
```
D:\logsieve/
├── index.html
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── postcss.config.js
├── src/
│   ├── main.tsx
│   ├── index.css
│   ├── App.tsx
│   ├── types.ts
│   ├── workers/
│   │   └── log-db.worker.ts     # Handles regex parsing and sql.js DB operations
│   ├── components/
│   │   ├── Dropzone.tsx         # File drop target
│   │   ├── AnalyticsPanel.tsx   # Left-side widgets and filters
│   │   ├── ConsoleViewer.tsx    # Right-side virtual log stream
│   │   └── DetailDrawer.tsx     # Popout panel for selected log lines
│   └── lib/
│       └── logParsers.ts        # Built-in regex configurations
```

### 2. SQL database Schema
The in-memory SQLite database will store logs in the following schema:
```sql
CREATE TABLE logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT,         -- ISO format timestamp or raw time
  epoch_ms INTEGER,       -- Unix timestamp in ms for fast sorting/timeline queries
  level TEXT,             -- ERROR, WARN, INFO, DEBUG
  method TEXT,            -- GET, POST, etc. (null if not web log)
  path TEXT,              -- Request path (null if not web log)
  status INTEGER,         -- HTTP response code (null if not web log)
  response_time INTEGER,  -- Response time in ms (null if not web log)
  message TEXT,           -- The core log message
  raw_line TEXT           -- The full raw input line
);

-- Indexes for fast filtering and analytics aggregations
CREATE INDEX idx_logs_level ON logs(level);
CREATE INDEX idx_logs_status ON logs(status);
CREATE INDEX idx_logs_epoch ON logs(epoch_ms);
```

### 3. Log Ingestion & Worker Communication
Communication between the Main React thread and the Web Worker:
```
Main UI Thread                     Web Worker (sql.js)
    |                                    |
    |---- file-selected (File Object)--->|
    |                                    | Parses line-by-line using regex templates
    |                                    | Creates & populates SQLite table
    |                                    |
    |<--- progress (percent, count)------| (Periodic update during indexing)
    |                                    |
    |<--- index-complete (total rows)---|
    |                                    |
    |---- run-query (SQL query string)-->| Runs SQL and fetches result rows
    |                                    |
    |<--- query-results (JSON rows)------|
```

## Verification Plan

### Automated Tests
*   **Parser Unit Tests**: Validate built-in regex parsers (Nginx Combined, Spring Boot, generic log) against test log strings using Vitest.
*   **Database Query Tests**: Verify SQLite insertion, indexing, and aggregations run correctly inside worker environments.

### Manual Verification
*   **Large File Test**: Load a 100,000-line server log file, verifying UI remains responsive (60fps scrolling) and ingestion completes within 5 seconds.
*   **Filter Accuracy**: Click `[ERROR]` and check that the log viewer shows only error lines and the charts update accordingly.
*   **Custom SQL Test**: Input custom SQL like `SELECT path, COUNT(*) as count FROM logs WHERE status = 404 GROUP BY path ORDER BY count DESC` and verify results are displayed in a clean tabular view.
