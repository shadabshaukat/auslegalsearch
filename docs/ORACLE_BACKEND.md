# Oracle 23ai Backend for AUSLegalSearch v3

This document explains how to run AUSLegalSearch with Oracle Autonomous Database 23ai as the storage backend, while preserving the existing Postgres-first core logic.

Overview
- Backend switch is controlled by environment variable AUSLEGALSEARCH_DB_BACKEND.
- Default remains Postgres. No behavior changes for existing setups.
- Oracle backend uses python-oracledb via SQLAlchemy and stores vectors using the native Oracle VECTOR data type. Similarity search runs SQL-side via vector_distance(), with optional APPROX to leverage HNSW/IVF vector indexes created through DBMS_VECTOR.

Switching Backends
- Postgres (default)
  - AUSLEGALSEARCH_DB_BACKEND=postgres
- Oracle
  - AUSLEGALSEARCH_DB_BACKEND=oracle

Environment Variables

Option A: Single DSN URL (recommended when possible)
- ORACLE_SQLALCHEMY_URL=oracle+oracledb://user:password@myadb_high

Option B: Individual fields
- ORACLE_DB_USER=your_db_user
- ORACLE_DB_PASSWORD=your_db_password
- ORACLE_DB_DSN=myadb_high            # TNS alias or EZConnect DSN
- ORACLE_WALLET_LOCATION=/path/to/wallet/dir  # Optional; sets TNS_ADMIN for Autonomous DB

Pool/Timeout tuning (shared with Postgres connector)
- AUSLEGALSEARCH_DB_POOL_SIZE=10
- AUSLEGALSEARCH_DB_MAX_OVERFLOW=20
- AUSLEGALSEARCH_DB_POOL_RECYCLE=1800
- AUSLEGALSEARCH_DB_POOL_TIMEOUT=30

Oracle AI Vector Search (optional index/bootstrap)
- AUSLEGALSEARCH_ORA_APPROX=1                    # use APPROX vector_distance() to enable index usage
- AUSLEGALSEARCH_ORA_AUTO_VECTOR_INDEX=0|1       # auto-create a vector index on embeddings.vector during bootstrap
- AUSLEGALSEARCH_ORA_INDEX_TYPE=HNSW|IVF
- AUSLEGALSEARCH_ORA_DISTANCE=COSINE|EUCLIDEAN|EUCLIDEAN_SQUARED|DOT|MANHATTAN|HAMMING
- AUSLEGALSEARCH_ORA_ACCURACY=90                 # target accuracy for approximate search
- AUSLEGALSEARCH_ORA_INDEX_PARALLEL=1            # parallelism for index build
- AUSLEGALSEARCH_ORA_HNSW_NEIGHBORS=16
- AUSLEGALSEARCH_ORA_HNSW_EFCONSTRUCTION=200
- AUSLEGALSEARCH_ORA_IVF_PARTITIONS=100

File Changes in this branch
- db/store.py: Dispatcher that exports the same symbols as before, selecting backend by AUSLEGALSEARCH_DB_BACKEND
- db/store_postgres.py: Previous Postgres models and helpers (pgvector + FTS)
- db/store_oracle.py: Oracle models and helpers (native VECTOR type + SQL vector_distance(); optional DBMS_VECTOR index auto-create)
- db/connector.py: Dispatcher for connectors
- db/connector_postgres.py: Previous Postgres connector
- db/connector_oracle.py: Oracle connector (python-oracledb via SQLAlchemy)

What works on Oracle (native VECTOR)
- Schema creation for core tables (users, documents, embeddings, sessions, etc.)
- Ingestion (documents + embeddings) using the same code paths
- Embeddings stored in Oracle native VECTOR(dim, FLOAT32, DENSE)
- Vector search: SQL-side vector_distance(e.vector, :qv) with optional APPROX for index usage
- Optional HNSW/IVF vector indexes via DBMS_VECTOR.CREATE_INDEX (env-gated)
- BM25-like search: case-insensitive LIKE over documents.content
- Hybrid search: blends SQL vector_distance and LIKE-based results
- FTS endpoint: LIKE-based fallback over documents and metadata JSON text

Postgres-only functionality (not in Oracle baseline)
- pgvector-based vector operators/indexes (<=>, IVFFLAT/HNSW)
- PostgreSQL FTS (tsvector/ts_headline) and trigram operators
- Post-load DDL and generated columns in schema-post-load/

Verification

1) Set environment
- export AUSLEGALSEARCH_DB_BACKEND=oracle
- export ORACLE_SQLALCHEMY_URL="oracle+oracledb://user:pass@myadb_high"
  or set ORACLE_DB_USER / ORACLE_DB_PASSWORD / ORACLE_DB_DSN (and ORACLE_WALLET_LOCATION if using wallet)

2) Ping and bootstrap
- The FastAPI/Gradio/Streamlit apps and ingestion workers will:
  - ping DB (SELECT 1 FROM dual)
  - create core tables automatically (if AUSLEGALSEARCH_AUTO_DDL=1)

3) Test minimal flows
- Run ingestion for a small folder to populate data
- Exercise /search/vector, /search/bm25, /search/hybrid, /search/fts
- Use Streamlit/Gradio UIs or FastAPI endpoints

Performance Notes
- The Oracle baseline ranks vectors client-side (Python). This is suitable for demos and smaller datasets.
- For production-scale vector search, plan a follow-up:
  - Use Oracle VECTOR column type and domain indexes
  - Replace client-side cosine with SQL-side vector operations
  - Add Oracle Text for full-text features comparable to PostgreSQL FTS

Troubleshooting
- oracledb package missing: pip install oracledb
- Wallet/TNS issues: verify ORACLE_WALLET_LOCATION and TNS_ADMIN; check connectivity using sqlplus or SQL Developer
- Permissions: ensure the schema user can create tables and run queries
- Large scans: reduce AUSLEGALSEARCH_ORA_VECTOR_SCAN_LIMIT, or migrate to Oracle VECTOR type and SQL ranking

Rollback / Staying on Postgres
- Simply unset or set AUSLEGALSEARCH_DB_BACKEND=postgres
- Postgres codepaths and performance features (pgvector, FTS, trigram) remain unchanged
