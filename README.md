# Hive Metastore Local Setup using Docker

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Docker](https://img.shields.io/badge/Docker-Required-blue?logo=docker)](https://www.docker.com/)
[![Hive](https://img.shields.io/badge/Apache%20Hive-4.0.0-yellow?logo=apache)](https://hive.apache.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-14-blue?logo=postgresql)](https://www.postgresql.org/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

> Run a fully functional Apache Hive Metastore locally in minutes using Docker — no manual Hadoop installation, no complex configuration, no cloud dependencies.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Features](#features)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Verify Setup](#verify-setup)
- [Connecting to Hive Metastore](#connecting-to-hive-metastore)
- [Configuration Reference](#configuration-reference)
- [Use Cases](#use-cases)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Project Overview

**Apache Hive Metastore** is the central metadata repository for the modern data lakehouse. It stores schema definitions, table locations, partition mappings, and storage format information — and is a required dependency for tools like **Apache Spark**, **Trino**, **Presto**, **Flink**, and **Dremio**.

Setting up Hive Metastore manually is notoriously painful:

- Requires installing and configuring Hadoop
- Requires a running relational database (PostgreSQL, MySQL)
- Requires correct JDBC driver placement
- Requires configuring `hive-site.xml` with dozens of properties
- Configuration drift between environments causes hard-to-debug failures

**This repository eliminates all of that.**

With a single `docker compose up -d`, you get a fully functional Hive Metastore backed by PostgreSQL and optionally connected to Google Cloud Storage (GCS) — ready for local development, integration testing, and Spark/Trino experimentation.

---

## Features

- **One-command startup** — `docker compose up -d` and you're running
- **Apache Hive 4.0.0** — latest stable release
- **PostgreSQL 14 backend** — production-grade metadata store with health checks
- **GCS connector pre-installed** — plug in your GCS bucket, no extra setup
- **AWS-ready** — commented AWS credential blocks for easy S3 switching
- **Environment-namespaced containers** — run multiple independent instances side by side
- **Configurable via `.env`** — no code changes needed to customize
- **Shared Docker network** — integrates cleanly with Spark, Trino, or any containerized stack
- **Schema auto-creation** — `datanucleus.schema.autoCreateAll` on first boot
- **Health checks built-in** — Docker reports true readiness, not just process start

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                  Docker Network                      │
│              (spark-shared-network)                  │
│                                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐ │
│  │   hive-metastore     │  │  postgres-{DEV}-      │ │
│  │   -{DEV}             │  │  metastore            │ │
│  │                      │  │                       │ │
│  │  Apache Hive 4.0.0   │◄─│  PostgreSQL 14        │ │
│  │  Thrift Server       │  │  metastore_{DEV} DB   │ │
│  │  Port: {BASE_PORT}83 │  │  Port: {BASE_PORT}32  │ │
│  │                      │  │                       │ │
│  │  + PostgreSQL JDBC   │  │  pgdata volume        │ │
│  │  + GCS Connector     │  │  (persistent)         │ │
│  └──────────┬───────────┘  └──────────────────────┘ │
└─────────────┼───────────────────────────────────────┘
              │
              ▼ Thrift Protocol (port 9083)
  ┌───────────────────────┐
  │  Spark / Trino /      │
  │  Presto / Flink       │
  │  (your local tools)   │
  └───────────────────────┘
              │
              ▼ Object Storage
  ┌───────────────────────┐
  │  GCS / S3 / Local FS  │
  │  (warehouse location) │
  └───────────────────────┘
```

**How it works:**

1. **PostgreSQL** starts first and passes a health check (`pg_isready`)
2. **Hive Metastore** waits for PostgreSQL to be healthy, then starts the Thrift server
3. On first boot, **DataNucleus** auto-creates the metastore schema in PostgreSQL
4. The Metastore exposes the standard **Thrift port 9083** on your host machine
5. Any tool that supports the Hive Metastore protocol can connect immediately

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Docker | 20.10+ | [Install Docker](https://docs.docker.com/get-docker/) |
| Docker Compose | v2.x | Bundled with Docker Desktop |
| GCS Bucket *(optional)* | — | Only needed for GCS warehouse |
| `gcloud` CLI *(optional)* | — | For local GCS auth via Application Default Credentials |

---

## Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/hive-metastore-setup.git
cd hive-metastore-setup
```

### 2. Configure Your Environment

Copy the example below into a `.env` file and update the values to match your environment:

```properties
# --- Infrastructure ---
HIVE_WAREHOUSE_DIR=gs://your-project-bucket/hive_meta

# --- Environment Naming ---
# Suffix used for container names (e.g., hive-metastore-dev)
DEV=dev

# Port prefix: BASE_PORT=60 → Metastore on 6083, Postgres on 6032
BASE_PORT=50

# --- Database Credentials ---
DB_USER=hive
DB_PASS=hive

# --- Docker Network ---
DOCKER_NETWORK=spark-shared-network
NETWORK_DRIVER=bridge
NETWORK_IS_EXTERNAL=false

# Uncomment to set a custom Metastore URI
# HIVE_METASTORE_URI=thrift://hive-metastore-dev:9083
```

> **Note:** If you are using GCS, ensure your Application Default Credentials are configured:
> ```bash
> gcloud auth application-default login
> ```
> Then uncomment the gcloud volume mount in `docker-compose.yml`.

### 3. Start the Services

```bash
docker compose up --build -d
```

Docker will:
- Pull the PostgreSQL 14 image
- Build the custom Hive Metastore image (downloads PostgreSQL JDBC + GCS connector)
- Start PostgreSQL and wait for it to be healthy
- Start Hive Metastore and initialize the schema on first boot

First build takes ~2–3 minutes due to JAR downloads. Subsequent starts are fast.

---

## Verify Setup

### Check Container Status

```bash
docker compose ps
```

Both services should show `healthy`:

```
NAME                          STATUS
hive-metastore-dev            healthy
postgres-dev-metastore        healthy
```

### Check Hive Metastore Logs

```bash
docker logs -f hive-metastore-dev
```

Look for this line to confirm successful startup:

```
Started metastore service
```

### Test the Thrift Port

```bash
# Quick TCP connectivity test
nc -zv localhost 5083    # if BASE_PORT=50
```

Or via Docker:

```bash
docker exec hive-metastore-dev bash -c \
  "timeout 1 bash -c 'cat < /dev/null > /dev/tcp/localhost/9083' && echo 'Metastore reachable'"
```

### Check PostgreSQL

```bash
docker exec -it postgres-dev-metastore \
  psql -U hive -d metastore_dev -c "\dt" | head -20
```

You should see Hive Metastore schema tables like `TBLS`, `DBS`, `PARTITIONS`, etc.

---

## Connecting to Hive Metastore

### Apache Spark (PySpark)

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("LocalHiveTest") \
    .config("spark.sql.catalogImplementation", "hive") \
    .config("hive.metastore.uris", "thrift://localhost:5083") \
    .config("spark.sql.warehouse.dir", "gs://your-bucket/hive_meta") \
    .enableHiveSupport() \
    .getOrCreate()

# Test it
spark.sql("SHOW DATABASES").show()
spark.sql("CREATE DATABASE IF NOT EXISTS test_db")
spark.sql("SHOW DATABASES").show()
```

### Apache Spark (spark-defaults.conf)

```properties
spark.sql.catalogImplementation=hive
spark.hadoop.hive.metastore.uris=thrift://localhost:5083
spark.sql.warehouse.dir=gs://your-bucket/hive_meta
```

### Trino / Presto

In your Trino catalog directory, create `hive.properties`:

```properties
connector.name=hive-hadoop2
hive.metastore.uri=thrift://localhost:5083
hive.s3.aws-access-key=YOUR_KEY
hive.s3.aws-secret-key=YOUR_SECRET
# Or for GCS:
# hive.gcs.json-key-file-path=/path/to/service-account.json
```

### Apache Flink (Table API)

```java
TableEnvironment tableEnv = TableEnvironment.create(settings);
tableEnv.registerCatalog("hive_catalog", new HiveCatalog(
    "hive_catalog",
    "default",
    "/path/to/hive-conf",  // directory containing hive-site.xml
    "4.0.0"
));
tableEnv.useCatalog("hive_catalog");
```

Create `hive-site.xml` for your Flink client pointing to:
```xml
<property>
  <name>hive.metastore.uris</name>
  <value>thrift://localhost:5083</value>
</property>
```

---

## Configuration Reference

### `.env` Variables

| Variable | Default | Description |
|---|---|---|
| `DEV` | `dev` | Environment suffix for container names |
| `BASE_PORT` | `50` | Port prefix (50 → Metastore:5083, PG:5032) |
| `HIVE_WAREHOUSE_DIR` | *(required)* | GCS/S3 path or local path for warehouse |
| `DB_USER` | `hive` | PostgreSQL username |
| `DB_PASS` | `hive` | PostgreSQL password |
| `DOCKER_NETWORK` | `spark-shared-network` | Docker network name |
| `NETWORK_DRIVER` | `bridge` | Docker network driver |
| `NETWORK_IS_EXTERNAL` | `false` | Set `true` to join existing network |
| `HIVE_METASTORE_URI` | *(auto-generated)* | Override Thrift URI |

### Running Multiple Environments

To run `dev` and `staging` simultaneously, use different `DEV` and `BASE_PORT` values:

```bash
# Terminal 1 - Dev environment
DEV=dev BASE_PORT=50 docker compose up -d

# Terminal 2 - Staging environment  
DEV=staging BASE_PORT=60 docker compose up -d
```

This creates isolated containers and ports for each environment.

---

## Use Cases

| Use Case | Description |
|---|---|
| **Local Spark Development** | Test Spark SQL queries against a real metastore without needing a cloud cluster |
| **Iceberg / Delta / Hudi Testing** | Validate table format integrations and schema evolution locally |
| **CI/CD Integration Testing** | Spin up a disposable metastore in CI pipelines for end-to-end tests |
| **Trino / Presto Development** | Build and test Hive-catalog-based queries before pushing to production |
| **Data Lakehouse Prototyping** | Prototype lakehouse architectures locally before cloud deployment |
| **Schema Migration Testing** | Test DDL migrations and schema changes safely |
| **Multi-Environment Development** | Run isolated dev/staging metastores side by side on the same machine |

---

## Troubleshooting

### Metastore container exits immediately

**Check logs:**
```bash
docker logs hive-metastore-dev
```

**Common cause:** PostgreSQL wasn't healthy when Metastore tried to connect. The health check should prevent this, but if you see JDBC connection errors, wait a moment and restart:
```bash
docker compose restart hive-metastore
```

### Port already in use

**Error:** `Bind for 0.0.0.0:5083 failed: port is already allocated`

**Fix:** Change `BASE_PORT` in `.env` to an unused prefix:
```properties
BASE_PORT=61   # Metastore → 6183, Postgres → 6132
```

### GCS authentication errors

**Error:** `401 Unauthorized` or `Could not load credentials`

**Fix:** Authenticate with Application Default Credentials:
```bash
gcloud auth application-default login
```

Then uncomment the gcloud volume mount in `docker-compose.yml`:
```yaml
volumes:
  - ~/.config/gcloud:/root/.config/gcloud
```

### Schema initialization errors on first boot

**Error:** `Table 'DBS' already exists` or similar DataNucleus errors

**Fix:** This usually means a partial init from a previous run. Clear the volume:
```bash
docker compose down -v   # removes pgdata volume
docker compose up --build -d
```

### Cannot connect from host machine

Verify the port mapping. With `BASE_PORT=50`, the host port is `5083`:
```bash
curl -v telnet://localhost:5083
# or
nc -zv localhost 5083
```

If behind a firewall, ensure the port is open for localhost connections.

---

## Contributing

Contributions are welcome! Here's how to get started:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-improvement`
3. Make your changes (documentation, bug fixes, new integrations)
4. Test your changes locally using `docker compose up --build`
5. Open a Pull Request with a clear description

**Ideas for contributions:**
- AWS S3 setup guide and example configuration
- Azure ADLS Gen2 connector integration
- Docker Compose override file for Spark + Metastore together
- GitHub Actions CI workflow for automated testing
- Kubernetes / Helm chart equivalent

Please open an issue before starting large changes so we can discuss the approach.

---

## License

This project is licensed under the **Apache License 2.0** — see the [LICENSE](LICENSE) file for details.

---

*If this project helped you, please consider giving it a ⭐ on GitHub. It helps others find the project and motivates continued development.*
