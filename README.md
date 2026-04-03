# IMDb Distributed Database

A distributed database system built on IMDb title data, demonstrating horizontal fragmentation, data replication, concurrency control, and failure recovery across multiple MySQL nodes.

## Overview

This project implements a three-node distributed database where IMDb title records are partitioned by type:

| Node | Role | Data |
|------|------|------|
| Node 1 (`node1-central`) | Central | All titles (full replica) |
| Node 2 (`node2-movies`) | Fragment | Movies only |
| Node 3 (`node3-nonmovies`) | Fragment | TV series, shorts, and all other non-movie titles |

A Flask backend coordinates reads/writes across nodes, enforces replication, and exposes a REST API consumed by a browser-based frontend.

## Features

- **Horizontal fragmentation** – titles are split across fragment nodes by `title_type`
- **Replication** – every write is propagated to all relevant nodes; the central node always holds the full dataset
- **Concurrency control** – configurable MySQL isolation levels (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`) with built-in test endpoints
- **Failure recovery** – failed replications are logged to a `transaction_log` table and retried automatically with configurable back-off
- **Web UI** – dashboard showing live node health, title browser, create/update/delete titles, transaction log viewer, concurrency test runner, and recovery test runner

## Tech Stack

- **Backend:** Python 3, Flask, mysql-connector-python
- **Database:** MySQL 8.0 (three independent instances)
- **Frontend:** Plain HTML/CSS/JavaScript
- **Infrastructure:** Docker Compose

## Project Structure

```
├── data/
│   └── process_imdb.py          # Script to extract & clean IMDb TSV data into CSV
├── init-scripts/
│   ├── create_schema.sql         # Creates titles and transaction_log tables
│   └── node1.sql                 # Loads the pre-processed CSV into Node 1
├── src/
│   ├── backend/
│   │   ├── app.py                # Flask application & API routes
│   │   ├── db_manager.py         # Connection management & query helpers
│   │   ├── initialize_data.py    # Fragment sync from central on startup
│   │   ├── route.py              # Frontend page routes
│   │   ├── requirements.txt
│   │   ├── Dockerfile
│   │   └── replication/
│   │       ├── replication_manager.py  # Insert / update / delete with replication
│   │       ├── transaction_logger.py   # Logs operations to transaction_log table
│   │       ├── recovery_handler.py     # Automatic retry of failed replications
│   │       └── concurrency_tester.py   # Concurrent read/write test helpers
│   └── frontend/
│       ├── pages/                # HTML templates (dashboard, browse, create, logs, tests)
│       └── public/css/           # Stylesheets
├── docker-compose.yml
└── .env.example
```

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/)

### 1. Prepare the IMDb data

Download `title.basics.tsv` from [IMDb datasets](https://datasets.imdbws.com/) and place it in the `data/` directory, then run:

```bash
cd data
python process_imdb.py
```

This produces `data/node1_all_titles.csv` (up to 40,000 non-adult titles from 2000 onward) which is loaded into Node 1 on first start.

### 2. Configure environment (optional)

Copy `.env.example` to `.env` and update the values if you are deploying to separate VMs:

```bash
cp .env.example .env
```

### 3. Start the stack

```bash
docker compose up --build
```

The backend waits for all three MySQL nodes to pass their health checks before accepting traffic.

### 4. Open the UI

Navigate to [http://localhost](http://localhost) in your browser.

## API Reference

### Titles

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/titles` | List titles (paginated; `?page=1&limit=20&type=movie`) |
| `GET` | `/titles/search` | Search by `q`, `year_from`, `year_to`, `type`, `genre` |
| `GET` | `/title/<tconst>` | Get a single title by IMDb ID |
| `POST` | `/title` | Create a new title (replicated to all nodes) |
| `PUT` | `/title/<tconst>` | Update a title (`?isolation=SERIALIZABLE`) |
| `DELETE` | `/title/<tconst>` | Delete a title |

### Health & Logs

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Node connection status |
| `GET` | `/logs` | Recent transaction log entries |
| `GET` | `/recovery/status` | Count of pending replications |
| `POST` | `/recovery/auto-retry` | Start or stop the automatic retry loop (`{"action": "start"\|"stop"}`) |

### Concurrency Tests

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/test/concurrent-read` | Multiple simultaneous reads |
| `POST` | `/test/read-write-conflict` | Concurrent read and write |
| `POST` | `/test/concurrent-write` | Concurrent writes to the same record |
| `POST` | `/test/isolation-levels` | Run a test across all four isolation levels |

### Failure Recovery Tests

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/test/failure/fragment-to-central` | Simulate fragment write succeeding but central replication failing |
| `POST` | `/test/failure/central-recovery` | Trigger Node 1 recovery of missed transactions |
| `POST` | `/test/failure/central-to-fragment` | Simulate central write succeeding but fragment replication failing |
| `POST` | `/test/failure/fragment-recovery` | Trigger fragment node recovery (`{"node": "node2"\|"node3"}`) |

### Admin

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/initialize-fragments` | Re-sync fragment nodes from central |
| `POST` | `/admin/reset-database` | **Destructive** – wipe and reinitialize all data |

## Database Schema

```sql
-- Titles (present on all nodes; fragment nodes hold a subset)
CREATE TABLE titles (
    tconst           VARCHAR(20) PRIMARY KEY,
    title_type       VARCHAR(20) NOT NULL,
    primary_title    VARCHAR(500) NOT NULL,
    start_year       INT,
    runtime_minutes  INT,
    genres           VARCHAR(100),
    last_updated     TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Replication audit log (present on all nodes)
CREATE TABLE transaction_log (
    log_id          INT AUTO_INCREMENT PRIMARY KEY,
    transaction_id  VARCHAR(50) NOT NULL,
    source_node     VARCHAR(10) NOT NULL,
    target_node     VARCHAR(10) NOT NULL,
    operation_type  ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    record_id       VARCHAR(20) NOT NULL,
    status          ENUM('SUCCESS', 'FAILED', 'PENDING') DEFAULT 'PENDING',
    retry_count     INT DEFAULT 0,
    max_retries     INT DEFAULT 3,
    query_text      TEXT,
    query_params    JSON,
    error_message   TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_retry_at   TIMESTAMP NULL,
    completed_at    TIMESTAMP NULL
);
```
