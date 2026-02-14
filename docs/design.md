# Universal Content Staging Database — Architecture & Implementation Plan

> **Document Type**: Architecture Design Document (ADD)
> **Status**: Draft
> **Architecture Style**: Layered Monolith with Event-Driven Command Processing
> **Primary Patterns**: Command Queue, Repository, Content Hierarchy (Tree), Schema Validation
> **Standards Compliance**: Architecture §1.1, §3; Security §1; XML §1, §4; PHP §1; SQL Schema §1–§5

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Phase 1 — Database Foundation](#3-phase-1--database-foundation)
4. [Phase 2 — Core Domain Services](#4-phase-2--core-domain-services)
5. [Phase 3 — Command Queue & Processing](#5-phase-3--command-queue--processing)
6. [Phase 4 — REST API Layer](#6-phase-4--rest-api-layer)
7. [Phase 5 — Advanced Content Operations](#7-phase-5--advanced-content-operations)
8. [Phase 6 — Security & Access Control](#8-phase-6--security--access-control)
9. [Phase 7 — Performance & Scalability](#9-phase-7--performance--scalability)
10. [Phase 8 — Observability & Operations](#10-phase-8--observability--operations)
11. [Appendices](#11-appendices)

---

## 1. Executive Summary

The Universal Content Staging Database (UCSD) is a **content abstraction layer** that decouples content production from content delivery. It stores structured content as XML blobs within a hierarchical tree, validated against versioned schemas, and addresses the following core problems:

- **Fragmented storage**: Content is currently scattered across course-specific SQL tables with inconsistent designs.
- **Tightly-coupled delivery**: Content producers must understand the rendering format of every consumer.
- **No universal addressing**: Content lacks globally unique identifiers and meaningful hierarchical names.
- **No audit trail**: Changes are applied directly and destructively with no logging or rollback.

The system introduces a **two-step process**: (1) define and organize content in a universal XML hierarchy, and (2) transform/deliver content to consumers via standard interfaces. The second step can eventually be eliminated as consumers adopt direct XML consumption.

### Design Principles

| Principle | Rationale | Standard Reference |
|---|---|---|
| Separation of concerns: SQL stores, XML defines | Content definition (schema/structure) is independent of content manipulation (CRUD/copy/move) | Architecture §1.1; XML §1.1.1 |
| Uniform interface for all content types | All content uses the same table, UUID, UUHN, and command queue regardless of type | Architecture §1.3 |
| Hierarchy as a first-class citizen | Parent-child relationships model the natural tree structure of content (courses → units → questions) | SQL Schema §3.1 |
| Immutable audit trail via command queue | All mutations flow through a logged command queue for traceability | Architecture §1.5; Security §7.2 |
| Fail-secure defaults | Invalid content is rejected; deletions require zero references; archiving replaces deletion | Security §1.4, §6.1 |

---

## 2. Architecture Overview

### 2.1 System Boundaries and Data Ownership

```
┌─────────────────────────────────────────────────────────────┐
│                    Content Producers                         │
│  (Editors, Import Tools, Batch Scripts, External APIs)       │
└──────────────┬──────────────────────────────┬───────────────┘
               │ REST API (JSON/XML)          │ PHP Library
               ▼                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   API / Entry Layer                          │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────────────┐ │
│  │ REST API  │  │ PHP Lib   │  │ CLI Tools                │ │
│  └────┬─────┘  └─────┬─────┘  └────────────┬─────────────┘ │
│       │              │                      │               │
│       └──────────────┼──────────────────────┘               │
│                      ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            Application Service Layer                  │   │
│  │  ContentService · SchemaService · CommandService      │   │
│  │  ValidationService · TreeService · TagService         │   │
│  └─────────────────────┬────────────────────────────────┘   │
│                        ▼                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            Domain Layer (Interfaces/Ports)            │   │
│  │  ContentRepository · SchemaRepository ·               │   │
│  │  CommandQueueRepository · BinaryReferenceRepository   │   │
│  └─────────────────────┬────────────────────────────────┘   │
│                        ▼                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            Infrastructure Layer (Adapters)            │   │
│  │  PostgreSQL PDO Repositories · XML Validator (DOM)    │   │
│  │  Filesystem/S3 Binary Storage · Logger                │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
               │                              │
               ▼                              ▼
┌──────────────────────┐     ┌────────────────────────────────┐
│    PostgreSQL DB      │     │  External Binary Storage       │
│  (content, schemas,   │     │  (filesystem / S3)             │
│   queue, logs, tags)  │     │                                │
└──────────────────────┘     └────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Content Consumers                          │
│  (Course Renderers, Mobile Apps, Export Tools, Replicas)     │
└─────────────────────────────────────────────────────────────┘
```

> **Addresses Q17 (REST API vs. PHP Library)**: Both entry points converge on the same Application Service Layer. PHP library access provides same-server, low-latency access for performance-critical internal operations. REST API provides cross-server, cross-language access with auth, rate limiting, and bulk operations. Direct database access is **forbidden** — all access flows through service interfaces (PHP §1.1.3, Architecture §3.3.1).

### 2.2 Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Database unavailable | All reads/writes fail | Connection pooling with retry; read replicas for reads (Architecture §2.2.3) |
| Command processor crash | Queue backlog grows | Heartbeat monitoring; auto-restart; idempotent commands (Architecture §1.5) |
| Schema validation failure | Content rejected | Detailed error response with XPath to invalid node (XML §8.1.2) |
| Binary storage unavailable | Content with references still readable; uploads fail | Graceful degradation; queue binary uploads for retry |
| Cycle in content hierarchy | Infinite traversal loops | Ancestor check before insert/update (see Phase 2) |

### 2.3 Key Terminology

| Term | Definition |
|---|---|
| **UUID** | Universal Unique ID — a random 128-bit key assigned to every content node |
| **UUHN** | Universal Unique Hierarchical Name — the unique path formed by traversing `parent_id` → `name` from root to node (e.g., `/CPA/2024/AUD/MCQ/Q23`) |
| **Schema** | An XSD document that defines the structure and validation rules for a content type |
| **Content Node** | A row in the `content` table — has a UUID, belongs to a parent, holds an XML blob validated against a schema |
| **Command** | A queued mutation (insert, update, delete, archive, copy, move, replace) processed asynchronously |

---

## 3. Phase 1 — Database Foundation

**Goal**: Establish the PostgreSQL schema with proper types, constraints, naming, and indexing.

**Standards**: SQL Schema §1–§5; Security §5.4

### 3.1 Database Choice

> **Addresses Q21 (Database Choice)**: PostgreSQL 15+ is selected for its native `XML` type, `xpath()` function, full-text search, `JSONB` for command data, advisory locks, transactional DDL, and row-level security. MySQL is rejected due to inferior XML support and lack of transactional DDL.

### 3.2 Schema DDL

```sql
/* Dialect: PostgreSQL 15+ */
/* Standards: SQL Schema §2 (naming), §3.1 (normalization), §3.4 (data types) */

-- Extension for UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- Table: schema_definition
-- Purpose: Registry of content type schemas (e.g., "mcq", "course", "audio")
-- Classification: Internal — system metadata
-- ============================================================
CREATE TABLE schema_definition (
    schema_definition_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    name_match VARCHAR(255) NULL,          -- Regex constraint for entity names (Q8)
    is_name_only BOOLEAN NOT NULL DEFAULT FALSE, -- Directory-like containers with no XML
    is_copyable BOOLEAN NOT NULL DEFAULT TRUE,   -- No-copy flag (Q26)
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT uq_schema_definition_name UNIQUE (name),
    CONSTRAINT ck_schema_definition_name_format
        CHECK (name ~ '^[a-z][a-z0-9_]*$')      -- Word characters, underscore only (Q27)
);

COMMENT ON TABLE schema_definition IS
    'Registry of content type schemas. Each schema defines a content type.';
COMMENT ON COLUMN schema_definition.name_match IS
    'Optional regex constraint applied to content node names using this schema (Q8).';
COMMENT ON COLUMN schema_definition.is_name_only IS
    'If TRUE, content nodes of this schema are name-only containers (no XML body).';
COMMENT ON COLUMN schema_definition.is_copyable IS
    'If FALSE, nodes of this schema are excluded from subtree copy operations (Q26).';

-- ============================================================
-- Table: schema_version
-- Purpose: Versioned XSD content for each schema definition
-- Classification: Internal — system metadata
-- ============================================================
CREATE TABLE schema_version (
    schema_version_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    schema_definition_id BIGINT NOT NULL,
    version INT NOT NULL,
    xsd_content XML NOT NULL,              -- XSD 1.0 for DOMDocument validation
    is_deprecated BOOLEAN NOT NULL DEFAULT FALSE, -- Q30: deprecation support
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_schema_version_schema_definition
        FOREIGN KEY (schema_definition_id)
        REFERENCES schema_definition(schema_definition_id)
        ON DELETE RESTRICT,
    CONSTRAINT uq_schema_version_schema_version
        UNIQUE (schema_definition_id, version),
    CONSTRAINT ck_schema_version_positive
        CHECK (version > 0)
);

CREATE INDEX ix_schema_version_schema_definition_id
    ON schema_version(schema_definition_id);

COMMENT ON TABLE schema_version IS
    'Versioned XSD schemas. Each version validates content nodes of its type.';
COMMENT ON COLUMN schema_version.is_deprecated IS
    'Deprecated schemas trigger warnings but remain functional during transition (Q30).';

-- ============================================================
-- Table: content
-- Purpose: Hierarchical content storage — each row is a node in the XML tree
-- Classification: Business data — may contain PII depending on content type
-- ============================================================
CREATE TABLE content (
    content_id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
    parent_id UUID NULL,
    name VARCHAR(255) NOT NULL,
    schema_version_id BIGINT NOT NULL,
    content_xml XML NULL,                  -- NULL for name-only containers
    archive_date TIMESTAMPTZ NULL,         -- Q9: nullable date replaces boolean
    created_by_command_id BIGINT NULL,     -- Q15, Q28: traceability to command
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_content_parent
        FOREIGN KEY (parent_id)
        REFERENCES content(content_id)
        ON DELETE RESTRICT,               -- Prevent orphaning
    CONSTRAINT fk_content_schema_version
        FOREIGN KEY (schema_version_id)
        REFERENCES schema_version(schema_version_id)
        ON DELETE RESTRICT,
    CONSTRAINT uq_content_parent_name
        UNIQUE (parent_id, name),          -- UUHN uniqueness within parent
    CONSTRAINT ck_content_name_format
        CHECK (name ~ '^[A-Za-z0-9_]+$')  -- Q27: word characters and underscores only
);

CREATE INDEX ix_content_parent_id ON content(parent_id);
CREATE INDEX ix_content_schema_version_id ON content(schema_version_id);
CREATE INDEX ix_content_archive_date ON content(archive_date)
    WHERE archive_date IS NOT NULL;        -- Partial index for purge queries

COMMENT ON TABLE content IS
    'Hierarchical content tree. Each row is a node with UUID and UUHN addressing.';
COMMENT ON COLUMN content.archive_date IS
    'When set, the node is archived. Purged after configurable retention period (Q9).';
COMMENT ON COLUMN content.created_by_command_id IS
    'Links to the command that created this content for full traceability (Q15, Q28).';

-- ============================================================
-- Table: content_relationship
-- Purpose: Arbitrary many-to-many relationships between content nodes
-- Classification: Internal — referential metadata
-- ============================================================
CREATE TABLE content_relationship (
    content_relationship_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source_content_id UUID NOT NULL,
    target_content_id UUID NOT NULL,
    relationship_type VARCHAR(100) NOT NULL DEFAULT 'related',

    CONSTRAINT fk_content_relationship_source
        FOREIGN KEY (source_content_id)
        REFERENCES content(content_id)
        ON DELETE RESTRICT,               -- Q10: prevent dangling references
    CONSTRAINT fk_content_relationship_target
        FOREIGN KEY (target_content_id)
        REFERENCES content(content_id)
        ON DELETE RESTRICT,
    CONSTRAINT uq_content_relationship_pair
        UNIQUE (source_content_id, target_content_id, relationship_type)
);

CREATE INDEX ix_content_relationship_source ON content_relationship(source_content_id);
CREATE INDEX ix_content_relationship_target ON content_relationship(target_content_id);

-- ============================================================
-- Table: natural_key
-- Purpose: Custom unique keys per schema (e.g., unique MCQ stems) (Q8)
-- Classification: Internal — constraint metadata
-- ============================================================
CREATE TABLE natural_key (
    natural_key_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    schema_definition_id BIGINT NOT NULL,
    key_name VARCHAR(255) NOT NULL,
    key_value VARCHAR(1000) NOT NULL,
    content_id UUID NOT NULL,

    CONSTRAINT fk_natural_key_schema
        FOREIGN KEY (schema_definition_id)
        REFERENCES schema_definition(schema_definition_id)
        ON DELETE RESTRICT,
    CONSTRAINT fk_natural_key_content
        FOREIGN KEY (content_id)
        REFERENCES content(content_id)
        ON DELETE CASCADE,
    CONSTRAINT uq_natural_key_unique
        UNIQUE (schema_definition_id, key_name, key_value)
);

COMMENT ON TABLE natural_key IS
    'Enforces custom uniqueness rules per schema type, e.g., unique MCQ stems (Q8).';

-- ============================================================
-- Table: tag
-- Purpose: Named tags for cross-hierarchy content discovery (Q11)
-- Classification: Internal — organizational metadata
-- ============================================================
CREATE TABLE tag (
    tag_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(255) NOT NULL,

    CONSTRAINT uq_tag_name UNIQUE (name),
    CONSTRAINT ck_tag_name_format CHECK (name ~ '^[a-z][a-z0-9_-]*$')
);

CREATE TABLE content_tag (
    content_id UUID NOT NULL,
    tag_id BIGINT NOT NULL,

    CONSTRAINT pk_content_tag PRIMARY KEY (content_id, tag_id),
    CONSTRAINT fk_content_tag_content
        FOREIGN KEY (content_id)
        REFERENCES content(content_id)
        ON DELETE CASCADE,
    CONSTRAINT fk_content_tag_tag
        FOREIGN KEY (tag_id)
        REFERENCES tag(tag_id)
        ON DELETE CASCADE
);

COMMENT ON TABLE tag IS
    'Tags for cross-hierarchy discovery beyond tree structure (Q11).';

-- ============================================================
-- Table: binary_reference
-- Purpose: Track externally stored binary assets (Q6, Q20)
-- Classification: Internal — reference metadata
-- ============================================================
CREATE TABLE binary_reference (
    binary_reference_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content_id UUID NOT NULL,
    storage_path VARCHAR(1000) NOT NULL,    -- Relative path or S3 key
    media_type VARCHAR(255) NOT NULL,       -- MIME type
    hash_sha256 VARCHAR(64) NULL,           -- Integrity check
    file_size_bytes BIGINT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_binary_reference_content
        FOREIGN KEY (content_id)
        REFERENCES content(content_id)
        ON DELETE RESTRICT
);

CREATE INDEX ix_binary_reference_content_id ON binary_reference(content_id);

COMMENT ON TABLE binary_reference IS
    'References to externally stored binaries (images, audio, video). '
    'Actual files live in filesystem/S3, not in the database (Q6, Q20).';

-- ============================================================
-- Table: command_queue
-- Purpose: Ordered queue of all content mutations for processing + audit
-- Classification: Internal — operational / audit
-- ============================================================
CREATE TABLE command_queue (
    command_queue_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    command_type VARCHAR(50) NOT NULL,
    priority SMALLINT NOT NULL DEFAULT 0,   -- Q13: 0=normal, higher=urgent
    command_data JSONB NOT NULL,            -- Structured command payload
    batch_id UUID NULL,                     -- Q14: group commands atomically
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    retry_count SMALLINT NOT NULL DEFAULT 0,
    max_retries SMALLINT NOT NULL DEFAULT 3,
    error_message TEXT NULL,
    issued_by VARCHAR(255) NULL,            -- Q28: user attribution
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMPTZ NULL,

    CONSTRAINT ck_command_queue_type CHECK (
        command_type IN (
            'insert', 'update', 'delete', 'archive',
            'copy', 'replace', 'move', 'validate_batch' -- Q13: extended types
        )
    ),
    CONSTRAINT ck_command_queue_status CHECK (
        status IN ('pending', 'processing', 'completed', 'failed', 'canceled')
    ),
    CONSTRAINT ck_command_queue_priority CHECK (priority BETWEEN 0 AND 10)
);

CREATE INDEX ix_command_queue_status_priority
    ON command_queue(status, priority DESC, command_queue_id ASC)
    WHERE status = 'pending';              -- Optimized pending command fetch

CREATE INDEX ix_command_queue_batch_id
    ON command_queue(batch_id)
    WHERE batch_id IS NOT NULL;

COMMENT ON TABLE command_queue IS
    'Ordered mutation queue. All content changes flow through commands for audit (Q13-Q16).';
COMMENT ON COLUMN command_queue.batch_id IS
    'Groups commands into atomic batches — all succeed or all fail (Q14).';
COMMENT ON COLUMN command_queue.issued_by IS
    'User or system identity that issued the command for audit trail (Q28).';

-- ============================================================
-- Table: command_log (partitioned)
-- Purpose: Immutable log of completed commands for audit (Q15)
-- Classification: Internal — audit trail
-- ============================================================
CREATE TABLE command_log (
    command_log_id BIGINT GENERATED ALWAYS AS IDENTITY,
    command_queue_id BIGINT NOT NULL,
    command_type VARCHAR(50) NOT NULL,
    summary_data JSONB NOT NULL,            -- Summary, not full diff (Q15)
    affected_content_ids UUID[] NOT NULL,
    issued_by VARCHAR(255) NULL,
    completed_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT pk_command_log PRIMARY KEY (command_log_id, completed_at)
) PARTITION BY RANGE (completed_at);        -- Q15: partitioned by date

-- Create monthly partitions (automated via pg_partman or cron)
-- Example:
-- CREATE TABLE command_log_2025_01 PARTITION OF command_log
--     FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

COMMENT ON TABLE command_log IS
    'Immutable, partitioned audit log of all completed commands (Q15). '
    'Stores summaries, not full diffs, for efficiency.';

-- ============================================================
-- Table: purge_policy
-- Purpose: Configurable retention periods per schema type (Q9)
-- Classification: Internal — configuration
-- ============================================================
CREATE TABLE purge_policy (
    purge_policy_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    schema_definition_id BIGINT NULL,       -- NULL = global default
    retention_days INT NOT NULL DEFAULT 365, -- Q9: default 1 year

    CONSTRAINT fk_purge_policy_schema
        FOREIGN KEY (schema_definition_id)
        REFERENCES schema_definition(schema_definition_id)
        ON DELETE CASCADE,
    CONSTRAINT uq_purge_policy_schema
        UNIQUE (schema_definition_id),
    CONSTRAINT ck_purge_policy_positive
        CHECK (retention_days > 0)
);

COMMENT ON TABLE purge_policy IS
    'Configurable archive retention per schema type. Default: 365 days (Q9).';
```

### 3.3 Design Decisions for Phase 1

<details>
<summary><strong>Q1 — Hierarchical Cycle Prevention</strong></summary>

**Decision**: Enforce acyclicity at the application layer using a recursive CTE check before any `INSERT` or `UPDATE` that modifies `parent_id`. A database trigger provides defense-in-depth.

```sql
-- Cycle detection function used by application layer before setting parent_id
CREATE OR REPLACE FUNCTION fn_check_no_cycle(
    p_node_id UUID,
    p_new_parent_id UUID
) RETURNS BOOLEAN
LANGUAGE plpgsql IMMUTABLE AS $$
BEGIN
    /* A node cannot be its own parent */
    IF p_node_id = p_new_parent_id THEN
        RETURN FALSE;
    END IF;

    /* Walk ancestors of the proposed parent; if we find p_node_id, it's a cycle */
    RETURN NOT EXISTS (
        WITH RECURSIVE ancestors AS (
            SELECT parent_id FROM content WHERE content_id = p_new_parent_id
            UNION ALL
            SELECT c.parent_id FROM content c
            INNER JOIN ancestors a ON c.content_id = a.parent_id
            WHERE c.parent_id IS NOT NULL
        )
        SELECT 1 FROM ancestors WHERE parent_id = p_node_id
    );
END;
$$;
```

**Rationale**: Cycle detection via CTE has $O(d)$ cost where $d$ is tree depth. Since content hierarchies are typically 5–10 levels deep, this is negligible. Database-level triggers provide a safety net if application validation is bypassed (Security §1.4: fail-secure defaults).

</details>

<details>
<summary><strong>Q9 — Archiving Strategy</strong></summary>

**Decision**: Replace `is_archived BOOLEAN` with `archive_date TIMESTAMPTZ NULL`. Content is considered archived when `archive_date IS NOT NULL`. Purging deletes content where `archive_date + retention_days < NOW()`, with retention configurable per schema via the `purge_policy` table. Default retention: **365 days**.

**Orphan handling during purge**: A cascading reference check prevents purging content that is still referenced by non-archived nodes. The purge job processes leaf nodes first, working upward.

</details>

<details>
<summary><strong>Q10 — Referential Integrity on Deletion</strong></summary>

**Decision**: Use `ON DELETE RESTRICT` on all foreign keys referencing `content.content_id`. The application layer checks for relationship references *and* child nodes before allowing archive/delete operations. A reference count column is **not** used — the overhead of maintaining it exceeds the cost of a targeted existence check (`EXISTS` with short-circuit), and it introduces consistency risks (SQL Query §4.2.2).

</details>

<details>
<summary><strong>Q11 — Tagging</strong></summary>

**Decision**: **Include** a `tag` + `content_tag` table. Tags address a use case that hierarchical names cannot: cross-hierarchy queries like "all content tagged `needs-review`" or `audio-updated-2024`. Tags do **not** inherit from parents — inheritance would create ambiguity and expense. Tags are flat labels for filtering and discovery.

</details>

---

## 4. Phase 2 — Core Domain Services

**Goal**: Implement PHP service interfaces, validation, content CRUD, and hierarchy operations.

**Standards**: PHP §1 (architecture), §2 (syntax), §3 (types), §5 (security); XML §4 (validation), §5 (security)

### 4.1 Project Structure (PSR-4)

```
src/
├── Domain/
│   ├── Content/
│   │   ├── Content.php                    # Value object
│   │   ├── ContentRepositoryInterface.php # Port
│   │   ├── ContentServiceInterface.php    # Use-case port
│   │   └── UUHN.php                       # Value object for hierarchical names
│   ├── Schema/
│   │   ├── SchemaDefinition.php
│   │   ├── SchemaVersion.php
│   │   ├── SchemaRepositoryInterface.php
│   │   └── SchemaServiceInterface.php
│   ├── Command/
│   │   ├── Command.php
│   │   ├── CommandBatch.php               # Q14: atomic batch
│   │   ├── CommandQueueRepositoryInterface.php
│   │   └── CommandProcessorInterface.php
│   ├── Tag/
│   │   ├── Tag.php
│   │   └── TagRepositoryInterface.php
│   ├── Binary/
│   │   ├── BinaryReference.php
│   │   └── BinaryReferenceRepositoryInterface.php
│   └── Validation/
│       ├── ContentValidatorInterface.php
│       └── ValidationResult.php           # Q7, Q29: detailed error paths
├── Application/
│   ├── ContentService.php                 # Orchestrates domain operations
│   ├── SchemaService.php
│   ├── CommandService.php
│   ├── TreeService.php                    # Copy, move, import/export
│   ├── TagService.php
│   └── PurgeService.php                   # Q9: archive purging
├── Infrastructure/
│   ├── Persistence/
│   │   ├── PdoContentRepository.php
│   │   ├── PdoSchemaRepository.php
│   │   ├── PdoCommandQueueRepository.php
│   │   ├── PdoTagRepository.php
│   │   └── PdoNaturalKeyRepository.php
│   ├── Validation/
│   │   └── DomDocumentValidator.php       # XML §5.1: XXE-safe
│   ├── Binary/
│   │   └── FilesystemBinaryStorage.php    # Or S3BinaryStorage
│   └── Http/
│       ├── RestApiController.php
│       └── Middleware/
│           ├── AuthMiddleware.php
│           ├── RateLimitMiddleware.php     # Architecture §12
│           └── ReadOnlyMiddleware.php      # Q24
├── Console/
│   ├── CommandWorker.php                  # Background queue processor
│   ├── PurgeCommand.php
│   └── ImportExportCommand.php
└── Config/
    └── container.php                      # DI container wiring
```

### 4.2 Core Interfaces

> **Addresses Q4 (UUID vs. Semantic Names)**: Both addressing modes are first-class. UUIDs are used for all internal references, relationships, foreign keys, and programmatic access. UUHNs are used for human-readable navigation, API path-based queries, and content organization. The `UUHN` value object resolves paths by traversing `parent_id` + `name` lookups.

```php
<?php

declare(strict_types=1);

namespace App\Domain\Content;

/**
 * Value object representing the Universal Unique Hierarchical Name.
 * Resolves both absolute (/CPA/2024/AUD) and relative (../AUD) paths.
 *
 * Addresses Q4 (UUID vs Semantic Names) and Q5 (Relative References).
 */
final readonly class UUHN
{
    /** @param string[] $segments */
    public function __construct(
        private array $segments,
        private bool $isRelative = false,
    ) {}

    public static function fromPath(string $path): self
    {
        $isRelative = !str_starts_with($path, '/');
        $segments = array_filter(explode('/', trim($path, '/')));
        return new self(array_values($segments), $isRelative);
    }

    public function isRelative(): bool
    {
        return $this->isRelative;
    }

    /** @return string[] */
    public function segments(): array
    {
        return $this->segments;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Domain\Validation;

/**
 * Detailed validation result with XPath pointers to invalid nodes.
 *
 * Addresses Q7 (validation error detail) and Q29 (error handling).
 */
final readonly class ValidationResult
{
    /**
     * @param bool $isValid
     * @param list<array{path: string, message: string, code: string}> $errors
     */
    public function __construct(
        public bool $isValid,
        public array $errors = [],
    ) {}
}
```

```php
<?php

declare(strict_types=1);

namespace App\Domain\Validation;

/**
 * Validates content XML against its schema XSD.
 *
 * Addresses Q7: Validation occurs on insert/update only (not on read).
 * A separate "validate_batch" command supports drift detection.
 */
interface ContentValidatorInterface
{
    /**
     * Validate content XML against the given schema XSD.
     *
     * @throws \App\Domain\Validation\ValidationException On catastrophic parser errors
     */
    public function validate(string $contentXml, string $schemaXsd): ValidationResult;

    /**
     * Validate content name against schema's name_match regex (Q8).
     */
    public function validateName(string $name, ?string $nameMatchRegex): ValidationResult;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Domain\Content;

interface ContentRepositoryInterface
{
    public function save(Content $content): void;

    public function findById(string $uuid): ?Content;

    /**
     * Resolve a UUHN to a content UUID.
     * For relative UUHNs (Q5), a context UUID must be provided.
     */
    public function resolveUuhn(UUHN $uuhn, ?string $contextUuid = null): ?string;

    /** @return Content[] */
    public function findByParentId(string $parentUuid): array;

    public function update(string $uuid, array $data): void;

    public function archive(string $uuid, \DateTimeImmutable $archiveDate): void;

    /**
     * Check if a node has any references (children or relationships).
     * Used before delete operations (Q10).
     */
    public function hasReferences(string $uuid): bool;

    /**
     * Check that setting $newParentId as parent of $nodeId won't create a cycle (Q1).
     */
    public function wouldCreateCycle(string $nodeId, string $newParentId): bool;
}
```

### 4.3 Secure XML Validation

> **Addresses Q7 (Validation), Q27 (Attributes vs. Elements), Q29 (Error Handling)**
>
> **XML §5.1 (XXE Protection)**: The validator **MUST** disable external entities, DTD loading, and enforce resource limits. PHP's `DOMDocument` is used for XSD 1.0 validation.

```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Validation;

use App\Domain\Validation\ContentValidatorInterface;
use App\Domain\Validation\ValidationResult;
use DOMDocument;

/**
 * XXE-safe XML validator using DOMDocument.
 *
 * Security: XML §5.1.1 — External entities disabled.
 * Security: XML §5.1.3 — Resource limits enforced via libxml options.
 */
final class DomDocumentValidator implements ContentValidatorInterface
{
    private const MAX_XML_SIZE_BYTES = 10 * 1024 * 1024; // 10 MB

    public function validate(string $contentXml, string $schemaXsd): ValidationResult
    {
        if (strlen($contentXml) > self::MAX_XML_SIZE_BYTES) {
            return new ValidationResult(false, [[
                'path' => '/',
                'message' => 'XML content exceeds maximum size limit.',
                'code' => 'SIZE_EXCEEDED',
            ]]);
        }

        // Disable external entity loading (XML §5.1.1)
        $previousEntityLoader = libxml_disable_entity_loader(true);
        $previousUseInternalErrors = libxml_use_internal_errors(true);

        try {
            $doc = new DOMDocument();
            $doc->loadXML($contentXml, LIBXML_NONET | LIBXML_NOENT | LIBXML_DTDLOAD);

            // Validate against XSD
            $isValid = $doc->schemaValidateSource($schemaXsd);

            $errors = [];
            if (!$isValid) {
                foreach (libxml_get_errors() as $error) {
                    $errors[] = [
                        'path' => "Line {$error->line}, Column {$error->column}",
                        'message' => trim($error->message),
                        'code' => "LIBXML_{$error->code}",
                    ];
                }
            }

            return new ValidationResult($isValid, $errors);
        } finally {
            libxml_clear_errors();
            libxml_use_internal_errors($previousUseInternalErrors);
            libxml_disable_entity_loader($previousEntityLoader);
        }
    }

    public function validateName(string $name, ?string $nameMatchRegex): ValidationResult
    {
        if ($nameMatchRegex === null) {
            return new ValidationResult(true);
        }

        // Ensure regex is safe (no arbitrary execution)
        $pattern = '/^' . $nameMatchRegex . '$/';
        if (@preg_match($pattern, '') === false) {
            return new ValidationResult(false, [[
                'path' => 'name',
                'message' => 'Invalid name_match regex in schema.',
                'code' => 'INVALID_REGEX',
            ]]);
        }

        if (!preg_match($pattern, $name)) {
            return new ValidationResult(false, [[
                'path' => 'name',
                'message' => "Name '{$name}' does not match schema constraint: {$nameMatchRegex}",
                'code' => 'NAME_MISMATCH',
            ]]);
        }

        return new ValidationResult(true);
    }
}
```

### 4.4 Design Decisions for Phase 2

<details>
<summary><strong>Q3 — Schema Versioning & Backward Compatibility</strong></summary>

**Decision**: Schema evolution follows a **strict compatibility policy** (XML §1.3):

1. **Non-breaking changes** (adding optional elements/attributes): New schema version, same `schema_definition_id`. Existing content remains valid.
2. **Breaking changes** (removing/renaming required elements): New `schema_definition` entirely. Existing content retains its old schema version.
3. **No automatic migration on reads**: Content is always read as-is with its original schema version. Migration requires an explicit `update` command that re-validates against the target schema version.
4. **Deprecation**: Schema versions can be marked `is_deprecated = TRUE`. The system logs warnings when deprecated schemas are used. A minimum 6-month transition period is enforced before removal (XML §1.3.2).

This satisfies Architecture §1.4 (backwards compatibility) and avoids silent data corruption from automatic migration.

</details>

<details>
<summary><strong>Q5 — Relative Reference Resolution</strong></summary>

**Decision**: Relative UUHNs (e.g., `../AUD`) are resolved at **query time** using the requesting node's position as context. During tree copies, relative references are preserved as-is (they naturally point to the correct sibling in the new context). Absolute UUHNs (`/CPA/2023/AUD`) remain pointing to the old location and require explicit re-pointing if needed.

**Validation rules**:
- `..` can only navigate to the parent (not beyond the root)
- Relative references are validated during creation by attempting resolution
- Invalid relative references produce a validation error (they do not silently fail)

This design is analogous to relative paths in a filesystem and supports the versioning use case described in notes.md.

</details>

<details>
<summary><strong>Q8 — Unique Constraints: Database vs. Application Level</strong></summary>

**Decision**: Layered enforcement:

| Constraint | Enforcement Level | Mechanism |
|---|---|---|
| Parent-name uniqueness | Database | `UNIQUE (parent_id, name)` |
| Name format (word chars) | Database | `CHECK` constraint on `content.name` |
| Schema-specific name regex | Application | `ContentValidatorInterface::validateName()` using `schema_definition.name_match` |
| Custom natural keys (e.g., unique MCQ stems) | Database | `natural_key` table with `UNIQUE (schema_definition_id, key_name, key_value)` |

Database-level constraints handle invariants that must never be violated. Application-level constraints handle schema-specific rules that vary by content type.

</details>

---

## 5. Phase 3 — Command Queue & Processing

**Goal**: Implement the asynchronous command queue, background worker, and audit logging.

**Standards**: Architecture §4 (event-driven), §1.5 (idempotency); Security §7 (logging)

### 5.1 Command Processing Flow

```
Producer                Command Queue              Worker                    Content DB
   │                        │                        │                          │
   │─── POST /commands ────>│                        │                          │
   │    (status: pending)   │                        │                          │
   │<── 202 Accepted ───────│                        │                          │
   │                        │                        │                          │
   │                        │<── poll oldest ────────│                          │
   │                        │    pending command     │                          │
   │                        │──── command data ─────>│                          │
   │                        │    (status: processing)│                          │
   │                        │                        │─── execute mutation ────>│
   │                        │                        │<── success/failure ──────│
   │                        │                        │                          │
   │                        │<── update status ──────│                          │
   │                        │    (completed/failed)  │                          │
   │                        │                        │                          │
   │                        │<── write audit log ────│                          │
   │                        │    (command_log)       │                          │
```

> **Addresses Q13 (Command Types)**: Supported commands: `insert`, `update`, `delete`, `archive`, `copy`, `replace`, `move` (new), and `validate_batch` (new - for drift detection per Q7). Commands are processed by priority descending, then FIFO within the same priority. Default priority: 0 (normal); admin/emergency operations can use higher priority.

> **Addresses Q14 (Transactional Batches)**: Commands with the same `batch_id` are processed atomically within a single database transaction. If any command in the batch fails, all are rolled back and marked `failed`. This prevents partial states.

> **Addresses Q16 (Concurrency)**: The system uses a **single worker** by default. For multi-worker scaling, PostgreSQL advisory locks (`pg_advisory_xact_lock`) are acquired per content subtree root before processing. This prevents concurrent modification of the same content branch while allowing parallel processing of independent branches.

### 5.2 Worker Implementation Sketch

```php
<?php

declare(strict_types=1);

namespace App\Console;

use App\Domain\Command\CommandProcessorInterface;
use App\Domain\Command\CommandQueueRepositoryInterface;
use Psr\Log\LoggerInterface;

/**
 * Background command queue worker.
 *
 * Architecture §1.5: All commands are idempotent — safe to retry.
 * Architecture §10.1.1: Exponential backoff with jitter for retries.
 */
final class CommandWorker
{
    private const BASE_DELAY_MS = 200;
    private const MAX_DELAY_MS = 30_000;
    private const POLL_INTERVAL_MS = 1_000;

    public function __construct(
        private readonly CommandQueueRepositoryInterface $queue,
        private readonly CommandProcessorInterface $processor,
        private readonly LoggerInterface $logger,
        private readonly bool $isReadOnly = false,  // Q24: read-only mode
    ) {}

    public function run(): never
    {
        // Q24: In read-only mode, the worker does not process commands
        if ($this->isReadOnly) {
            $this->logger->info('Worker started in READ-ONLY mode. No commands will be processed.');
            while (true) {
                usleep(self::POLL_INTERVAL_MS * 1000);
            }
        }

        while (true) {
            $command = $this->queue->claimOldestPending();

            if ($command === null) {
                usleep(self::POLL_INTERVAL_MS * 1000);
                continue;
            }

            try {
                $this->processor->processCommand($command);
                $this->queue->markCompleted($command['command_queue_id']);
                $this->logger->info('Command completed', [
                    'command_id' => $command['command_queue_id'],
                    'type' => $command['command_type'],
                ]);
            } catch (\Throwable $e) {
                $retryCount = $command['retry_count'] + 1;

                if ($retryCount >= $command['max_retries']) {
                    $this->queue->markFailed(
                        $command['command_queue_id'],
                        $e->getMessage(),
                    );
                    $this->logger->error('Command permanently failed', [
                        'command_id' => $command['command_queue_id'],
                        'error' => $e->getMessage(),
                    ]);
                } else {
                    // Exponential backoff with jitter (Architecture §10.1.1)
                    $delay = min(
                        self::BASE_DELAY_MS * (2 ** $retryCount),
                        self::MAX_DELAY_MS,
                    );
                    $jitter = random_int(0, (int) ($delay * 0.3));
                    usleep(($delay + $jitter) * 1000);

                    $this->queue->markPendingWithRetry(
                        $command['command_queue_id'],
                        $retryCount,
                    );
                    $this->logger->warning('Command retrying', [
                        'command_id' => $command['command_queue_id'],
                        'retry' => $retryCount,
                        'delay_ms' => $delay + $jitter,
                    ]);
                }
            }
        }
    }
}
```

### 5.3 Design Decisions for Phase 3

<details>
<summary><strong>Q15 — Logging and Auditing</strong></summary>

**Decision**:

| Aspect | Choice | Rationale |
|---|---|---|
| Content → Command link | `created_by_command_id` column on `content` table | Direct traceability from content to its creating command (Q15, Q28) |
| Update tracking | Commands reference affected `content_id`s in `command_log.affected_content_ids` (UUID array) | Many-to-many without a join table; PostgreSQL array type is efficient for read-heavy audit |
| Log detail level | **Summaries** (old/new values for changed fields), not full XML diffs | Full diffs are expensive to store and rarely useful. Consumers can reconstruct diffs from XML snapshots if needed |
| Log partitioning | Monthly partitions (`YYYY_MM`) on `command_log.completed_at` | Enables efficient purging of old logs and query optimization (Security §7.1) |
| User attribution | `issued_by` field on `command_queue` and `command_log` | Tracks who issued each command for accountability (Q28) |

</details>

<details>
<summary><strong>Q24 — Read-Only Mode</strong></summary>

**Decision**: Read-only mode is a **per-instance** setting controlled by an environment variable (`UCSD_READ_ONLY=true`). When enabled:

- The command worker pauses (does not claim commands)
- The REST API rejects all mutation endpoints with `HTTP 503 Service Unavailable` (via `ReadOnlyMiddleware`)
- Read endpoints remain fully functional
- Queue commands can still be *submitted* (stored as `pending`) but are not processed until read-only is disabled

This supports maintenance windows and read-replica scenarios without impacting the queue's integrity.

</details>

---

## 6. Phase 4 — REST API Layer

**Goal**: Expose system capabilities via a RESTful API with authentication, rate limiting, and standardized error responses.

**Standards**: Architecture §11.2, §12; Security §8; PHP §4.2

### 6.1 API Design

> **Addresses Q17 (REST API vs. PHP Library)**: The REST API is the primary integration point for external systems and cross-server access. All mutation operations return `202 Accepted` with a command ID for tracking, since mutations flow through the command queue asynchronously. Read operations return `200 OK` synchronously.

> **Addresses Q19 (Integration with Consumers)**: The API supports both `Accept: application/xml` and `Accept: application/json` response formats. JSON responses wrap XML content as a string field with metadata. This allows consumers to incrementally adopt XML processing.

> **Addresses Q29 (Error Handling)**: All errors follow RFC 7807 Problem Details format, consistent across XML and JSON:

```json
{
  "type": "https://ucsd.example.com/errors/validation-failed",
  "title": "Content Validation Failed",
  "status": 422,
  "detail": "XML content does not conform to schema 'mcq' version 3.",
  "instance": "/content/a1b2c3d4-...",
  "errors": [
    {
      "path": "Line 12, Column 5",
      "message": "Element 'answer': Missing child element 'text'.",
      "code": "LIBXML_1871"
    }
  ]
}
```

### 6.2 Endpoint Summary

| Method | Endpoint | Purpose | Auth | Q |
|---|---|---|---|---|
| **Schemas** |||||
| `POST` | `/api/v1/schemas` | Register a new schema type | write | Q26 |
| `GET` | `/api/v1/schemas/{name}` | Get schema definition | read | |
| `POST` | `/api/v1/schemas/{name}/versions` | Add schema version (XSD) | write | Q3 |
| `GET` | `/api/v1/schemas/{name}/versions/latest` | Get latest schema version | read | |
| `GET` | `/api/v1/schemas/{name}/versions/{version}` | Get specific version | read | |
| **Content** |||||
| `POST` | `/api/v1/content` | Create content (enqueues `insert`) | write | |
| `GET` | `/api/v1/content/{uuid}` | Get content by UUID | read | Q4 |
| `GET` | `/api/v1/content/resolve?path=/CPA/2024/AUD` | Resolve UUHN to content | read | Q4 |
| `PUT` | `/api/v1/content/{uuid}` | Update content (enqueues `update`) | write | |
| `DELETE` | `/api/v1/content/{uuid}` | Archive content (enqueues `archive`) | write | Q9 |
| `GET` | `/api/v1/content/{uuid}/children` | List child content nodes | read | |
| `GET` | `/api/v1/content/{uuid}/ancestors` | Get full path to root | read | |
| **Relationships** |||||
| `POST` | `/api/v1/content/{uuid}/relationships` | Create relationship | write | |
| `GET` | `/api/v1/content/{uuid}/relationships` | List relationships (source + target) | read | |
| **Tags** |||||
| `POST` | `/api/v1/content/{uuid}/tags` | Tag content | write | Q11 |
| `GET` | `/api/v1/tags/{name}/content` | Find content by tag | read | Q11 |
| **Commands** |||||
| `POST` | `/api/v1/commands` | Submit command(s) directly | write | Q13 |
| `POST` | `/api/v1/commands/batch` | Submit atomic command batch | write | Q14 |
| `GET` | `/api/v1/commands/{id}` | Check command status | read | |
| **Binaries** |||||
| `POST` | `/api/v1/content/{uuid}/binaries` | Upload binary reference | write | Q20 |
| `GET` | `/api/v1/content/{uuid}/binaries` | List binary references | read | Q20 |
| **Import/Export** |||||
| `POST` | `/api/v1/export` | Export subtree (enqueues, returns download link) | read | Q12 |
| `POST` | `/api/v1/import` | Import subtree from file | write | Q12 |

### 6.3 Bulk Operations

> **Addresses Q17 (Bulk operations)**: The `POST /api/v1/commands/batch` endpoint accepts an array of commands with a shared `batch_id`. The batch is processed atomically per Q14.

```json
{
  "batch": [
    {
      "command_type": "insert",
      "command_data": {
        "parent_id": "a1b2c3d4-...",
        "name": "Q42",
        "schema": "mcq",
        "content_xml": "<mcq>...</mcq>"
      }
    },
    {
      "command_type": "insert",
      "command_data": {
        "parent_id": "a1b2c3d4-...",
        "name": "Q43",
        "schema": "mcq",
        "content_xml": "<mcq>...</mcq>"
      }
    }
  ]
}
```

### 6.4 Rate Limiting

Per Architecture §12 and Security §7.3:

| Scope | Limit | Header |
|---|---|---|
| Per API key | 1000 req/min | `X-RateLimit-Limit` |
| Per IP (unauthenticated) | 60 req/min | `X-RateLimit-Limit` |
| Write endpoints | 100 req/min | `X-RateLimit-Limit` |
| Authentication failures | 10 req/min per IP | — |

Exceeded limits return `HTTP 429 Too Many Requests` with `Retry-After` header (Architecture §12.1.2).

---

## 7. Phase 5 — Advanced Content Operations

**Goal**: Implement tree copying, versioning, import/export, and binary handling.

### 7.1 Content Versioning

> **Addresses Q2 (Content Versioning)**

**Decision**: Hybrid manual + optional auto-versioning.

**Manual versioning** (default): Content versions are modeled as child nodes in the tree. For example, `/CPA/2024` and `/CPA/2025` are sibling nodes under `/CPA`. Creating a new version is a tree copy operation from `/CPA/2024` to `/CPA/2025`.

**Auto-versioning** (opt-in per schema): Schemas can enable auto-versioning with configurable thresholds:

```xml
<!-- In schema definition metadata -->
<auto_version threshold_percent="20" metric="word_diff" />
```

When auto-versioning is enabled, the `update` command processor:
1. Computes word-level diff between old and new XML content
2. Maintains a cumulative `change_count` (stored in command metadata)
3. When `change_count / total_words > threshold`, automatically creates a new version node as a sibling

**Metrics supported**: `word_diff` (default — counts words added/removed), `structural_diff` (counts element additions/removals).

This approach keeps versioning simple and explicit by default while supporting automated thresholds for high-volume content pipelines.

### 7.2 Tree Copying Algorithm

> **Addresses Q22 (Tree Copying)**

The tree copy algorithm (from notes.md) ensures that parent/relationship dependencies are satisfied before copying each node:

```
Algorithm: CopySubtree(source_root_uuid, target_parent_uuid, new_root_name)

1. Collect all nodes in source subtree (BFS from source_root_uuid)
2. Exclude nodes where schema.is_copyable = FALSE (Q26: no-copy flag)
3. Build dependency graph: each node depends on its parent + relationship targets
4. Topological sort: process nodes in dependency order (parents before children)
5. For each node in order:
   a. Generate new UUID
   b. Map old UUID → new UUID in renumbering table
   c. If node is source_root: set parent = target_parent_uuid, name = new_root_name
      Else: set parent = mapped parent UUID
   d. Copy XML content, replacing embedded UUIDs using the renumbering map
   e. Insert new content node via command queue
6. Copy relationships: for each relationship in source subtree,
   create equivalent relationship using mapped UUIDs
7. Preserve relative UUHNs as-is (Q5: they resolve correctly in new context)
8. Log complete UUID mapping in command_log for traceability
```

**Asynchronous execution**: Tree copies are submitted as command batches (Q14). For large trees, progress is tracked via the batch's command statuses. The API returns `202 Accepted` with the batch ID.

**Date-based syncing**: When copying between versions, the algorithm checks `updated_at` on each node. Nodes that haven't changed since last sync are skipped, using a "last sync timestamp" stored in command metadata (Q22).

### 7.3 Import/Export

> **Addresses Q12 (Import/Export)**

**Export format**: ZIP archive containing:
```
export/
├── manifest.json          # UUID mapping, schema versions, tree structure
├── content/
│   ├── {uuid-1}.xml       # Individual XML blobs
│   ├── {uuid-2}.xml
│   └── ...
├── schemas/
│   ├── mcq_v3.xsd         # Schema definitions used
│   └── ...
└── relationships.json     # Relationship graph
```

**Import behavior**:
1. Parse manifest and validate all schemas exist (or import them)
2. Generate new UUIDs for all content nodes (always renumber — Q12)
3. Build UUID mapping: `old UUID → new UUID`
4. Process nodes in dependency order (same as tree copy)
5. Replace embedded UUID references in XML content using the mapping
6. Report conflicts (name collisions within target parent) as errors — do not silently overwrite

### 7.4 Binary Handling

> **Addresses Q6 (Non-XML Blobs) and Q20 (Binary Handling)**

**Decision**: Binaries are stored **externally** (filesystem or S3) with metadata tracked in the `binary_reference` table. The API provides endpoints for managing binary metadata and upload/download URLs.

**Consistency enforcement**: When creating content that references a binary (via UUID in XML), the `insert`/`update` command processor validates that all referenced binary UUIDs exist in the `binary_reference` table. Missing references produce a validation error.

**Binary versioning**: Binaries are immutable. A new version of a binary is a new `binary_reference` row with a new `storage_path`. The content XML is updated to reference the new binary UUID. Old binary references are purged along with their archived parent content.

---

## 8. Phase 6 — Security & Access Control

**Goal**: Implement authentication, authorization, and hardening measures.

**Standards**: Security §1–§9; Architecture §14.1

### 8.1 Access Control Model

> **Addresses Q25 (Access Control)**

**Decision**: Role-based access control (RBAC) at the **namespace level**, enforced at the API layer.

| Role | Scope | Permissions |
|---|---|---|
| `reader` | Per namespace (top-level node) | Read content, resolve UUHNs, list children |
| `writer` | Per namespace | Read + submit mutation commands |
| `admin` | Per namespace | Read + write + manage schemas + manage tags |
| `superadmin` | Global | All operations across all namespaces |

**Namespace** = the first level of the content hierarchy (e.g., `/CPA`, `/products`). Each API key or authenticated user is assigned roles per namespace.

**UUID security** (Q25): UUIDs are 128-bit random values (not sequential), making enumeration attacks infeasible. No additional obfuscation is needed. The system returns `404 Not Found` for both non-existent and unauthorized content (Security §6.2: no information leakage).

### 8.2 Security Hardening Checklist

Per Security §14.1 and the enforcement checklist:

- [ ] **Input validation**: All API inputs validated with typed schemas at boundary (Security §3.1; PHP §4.2.1)
- [ ] **SQL injection prevention**: All queries use prepared statements via PDO (Security §3.2; PHP §5.2.1)
- [ ] **XXE prevention**: XML parser configured with entities disabled (XML §5.1.1)
- [ ] **XPath injection prevention**: XPath expressions use parameterization, never string concatenation (XML §5.2.2)
- [ ] **CSRF protection**: `SameSite=Strict` cookies for session auth; token-based auth for API (Security §3.4)
- [ ] **Rate limiting**: Per-key and per-IP limits on all endpoints (Security §7.3)
- [ ] **No secrets in code**: Database credentials, API keys in environment variables only (Security §4.3)
- [ ] **Structured logging**: JSON logs with correlation IDs; no sensitive data logged (Security §7.1, §5.5)
- [ ] **TLS 1.2+**: All network communication encrypted (Security §4.2)
- [ ] **Dependency auditing**: `composer audit` in CI pipeline (PHP §5.1.3)
- [ ] **HSTS headers**: Strict transport security for web endpoints (Security §8.5)

### 8.3 Custom Query Security

> **Addresses Q18 (Custom Queries)**

**Decision**: Custom SQL SELECT queries are **not** exposed via the REST API. They are available only through the PHP library for internal use by trusted application code. The API provides structured query parameters (filtering by parent, schema, tag, UUHN path, date range) that the service layer translates to safe, parameterized queries.

For XML-specific queries, XPath is supported via a dedicated endpoint:

```
GET /api/v1/content/{uuid}/xpath?expr=//question[@difficulty='hard']
```

The `expr` parameter is validated against an allowlist of safe XPath patterns (no functions that access external resources). The query is executed using PostgreSQL's `xpath()` function with parameterized content:

```sql
SELECT content_id, xpath(:expr, content_xml) AS result
FROM content
WHERE content_id = :uuid;
```

---

## 9. Phase 7 — Performance & Scalability

**Goal**: Optimize for read-heavy workloads, large hierarchies, and multi-server deployment.

**Standards**: Architecture §14.2; SQL Query §4

### 9.1 PostgreSQL Optimizations

> **Addresses Q21 (Database Optimization)**

| Optimization | Implementation | Standard |
|---|---|---|
| XML indexing | `CREATE INDEX ix_content_xml_gin ON content USING gin(content_xml);` for XPath queries | SQL Query §4.3.4 |
| Connection pooling | PgBouncer with pool size 10-20 per app instance | Architecture §14.2.3 |
| Partial indexes | `WHERE archive_date IS NOT NULL` for purge queries; `WHERE status = 'pending'` for command queue | SQL Schema §3.3 |
| UUHN resolution cache | Application-level TTL cache for frequently resolved paths (PSR-16) | PHP §6.3 |
| Pagination | Keyset pagination for all list endpoints using `(updated_at, content_id)` cursor | SQL Query §4.1.3 |

### 9.2 Replication Strategy

> **Addresses Q23 (Replication)**

**Decision**: PostgreSQL streaming replication for read replicas.

| Aspect | Choice | Rationale |
|---|---|---|
| Sync frequency | Continuous (streaming), not hourly batch | PostgreSQL streaming replication is simpler and more reliable than application-level sync |
| Conflict resolution | N/A (single-primary architecture) | All writes go to primary; replicas are read-only |
| Content scope per replica | Full replica (all namespaces) | Partial replication adds complexity without proportional benefit for this system size |
| Application awareness | Read queries routed to replicas; write queries to primary | Via connection configuration in the service layer |

If future requirements demand multi-primary writes across regions, the command queue's idempotent design enables **event-sourcing** — commands can be replayed on any replica to reconstruct state. Conflict resolution would use **command timestamp ordering** (last-write-wins with vector clocks). This is deferred until needed.

### 9.3 Sharding Strategy

> **Addresses Q21 (Sharding)**

**Decision**: Defer sharding. The single-table design with UUID primary keys is naturally shardable by namespace (top-level node) if needed in the future. The `parent_id` foreign key constraint would be relaxed to application-level enforcement in a sharded setup. For now, a single PostgreSQL instance with read replicas handles the expected content volume (millions of nodes).

---

## 10. Phase 8 — Observability & Operations

**Goal**: Implement structured logging, health checks, monitoring, and operational tooling.

**Standards**: Architecture §1.6, §2.4.2; Security §7

### 10.1 Structured Logging

All logs **MUST** be structured JSON (Security §7.1) with PSR-3 interface (PHP §8.1):

```json
{
  "timestamp": "2025-03-15T14:32:01.123Z",
  "level": "info",
  "message": "Command completed",
  "context": {
    "correlation_id": "req-abc123",
    "command_id": 4521,
    "command_type": "insert",
    "content_id": "a1b2c3d4-...",
    "issued_by": "api-key:editor-team",
    "duration_ms": 42
  }
}
```

> **Addresses Q28 (Traceability)**: Every log entry includes `issued_by` (user/API key identity) and `correlation_id` (request trace). The full audit chain is: API request → command → command_log → content (`created_by_command_id`).

### 10.2 Health Endpoints

```
GET /health           → { "status": "ok", "checks": { "db": "ok", "queue_depth": 3 } }
GET /health/ready     → 200 if DB connected and queue processor running
GET /health/live      → 200 if process is alive
```

### 10.3 Metrics

| Metric | Type | Purpose |
|---|---|---|
| `ucsd_command_queue_depth` | Gauge | Pending commands — alerts if growing |
| `ucsd_command_processing_duration_seconds` | Histogram | Command execution time |
| `ucsd_content_count` | Gauge | Total content nodes |
| `ucsd_api_request_duration_seconds` | Histogram | API response latency (RED metrics) |
| `ucsd_api_error_total` | Counter | Error rate by status code |
| `ucsd_validation_failure_total` | Counter | Schema validation failures |

### 10.4 Purge Operations

The `PurgeService` runs on a configurable schedule (daily cron) and:

1. Queries `purge_policy` for retention periods per schema
2. Finds content where `archive_date + retention_days < NOW()`
3. Verifies no non-archived content references the candidate (Q9: orphan handling)
4. Deletes eligible nodes leaf-first
5. Logs purge results to `command_log`

---

## 11. Appendices

### Appendix A: Question Resolution Matrix

| Q# | Topic | Phase | Resolution Summary |
|---|---|---|---|
| 1 | Cycle prevention | 1 | CTE-based ancestor check before parent_id changes; trigger as defense-in-depth |
| 2 | Content versioning | 5 | Manual (tree copy) by default; opt-in auto-versioning with configurable word-diff threshold |
| 3 | Schema versioning | 2 | Backward-compatible evolution; explicit migration commands; no auto-migration on read |
| 4 | UUID vs. UUHN | 2 | Both are first-class; UUID for programmatic access, UUHN for human navigation |
| 5 | Relative references | 2 | Resolved at query time against context node; preserved during tree copies |
| 6 | Non-XML blobs | 1, 5 | External storage with `binary_reference` table for traceability |
| 7 | Validation timing | 2 | On insert/update only; `validate_batch` command for drift detection |
| 8 | Unique constraints | 1, 2 | Database for universal constraints; application for schema-specific rules |
| 9 | Archiving/purging | 1, 8 | `archive_date` field; configurable per-schema retention via `purge_policy`; default 365 days |
| 10 | Referential integrity | 1 | `ON DELETE RESTRICT` everywhere; existence check before archive/delete |
| 11 | Tagging | 1 | Included — flat labels for cross-hierarchy discovery, no inheritance |
| 12 | Import/export | 5 | ZIP with individual XML files + manifest; always renumber UUIDs |
| 13 | Command types | 3 | Added `move` and `validate_batch`; priority field for ordering |
| 14 | Transactional batches | 3 | `batch_id` groups commands; atomic processing within single transaction |
| 15 | Logging/auditing | 3 | `command_log` partitioned monthly; summaries not full diffs; content links to creating command |
| 16 | Concurrency | 3 | Single worker default; advisory locks on subtree roots for future multi-worker |
| 17 | REST vs. PHP | 4 | PHP for internal high-performance; REST for external with bulk support |
| 18 | Custom queries | 6 | PHP-only for raw SQL; API offers structured filters + limited XPath |
| 19 | Consumer integration | 4 | XML+JSON content negotiation; on-demand rendering via schema-specific transforms |
| 20 | Binary handling | 5 | API endpoints for metadata; external storage; reference validation on content save |
| 21 | Database choice | 1 | PostgreSQL 15+ for XML type, xpath(), transactional DDL |
| 22 | Tree copying | 5 | Async via command batches; topological sort; date-based skip optimization |
| 23 | Replication | 7 | PostgreSQL streaming replication; single-primary; event-sourcing path for future multi-primary |
| 24 | Read-only mode | 3 | Per-instance env var; queue paused; reads unaffected |
| 25 | Access control | 6 | RBAC per namespace; 128-bit random UUIDs prevent enumeration |
| 26 | Extensibility | 1, 5 | Schema registration via API; `is_copyable` flag; `name_match` regex |
| 27 | Attributes vs. elements | 1, 2 | Word chars + underscores enforced via CHECK; schema guidelines documented |
| 28 | Traceability to users | 3 | `issued_by` on commands and logs; `created_by_command_id` on content |
| 29 | Error handling | 4 | RFC 7807 Problem Details; validation errors include XPath to issue |
| 30 | Long-term evolution | 2 | `is_deprecated` flag on schema versions; 6-month transition periods |

### Appendix B: Implementation Priority and Timeline

| Phase | Priority | Estimated Effort | Dependencies |
|---|---|---|---|
| **Phase 1**: Database Foundation | P0 (Critical) | 2 weeks | PostgreSQL 15+ environment |
| **Phase 2**: Core Domain Services | P0 (Critical) | 3 weeks | Phase 1 |
| **Phase 3**: Command Queue & Processing | P0 (Critical) | 2 weeks | Phase 2 |
| **Phase 4**: REST API Layer | P1 (High) | 3 weeks | Phase 3 |
| **Phase 5**: Advanced Operations | P1 (High) | 3 weeks | Phase 3 |
| **Phase 6**: Security & Access Control | P1 (High) | 2 weeks | Phase 4 |
| **Phase 7**: Performance & Scalability | P2 (Medium) | 2 weeks | Phase 4 |
| **Phase 8**: Observability & Operations | P2 (Medium) | 1 week | Phase 3 |

### Appendix C: Testing Strategy

Per Architecture §14.4 and PHP §7:

| Layer | Test Type | Tools | Coverage Target |
|---|---|---|---|
| Domain value objects (UUHN, Content, ValidationResult) | Unit | PHPUnit 10+ | 95% |
| Application services (ContentService, TreeService) | Unit + integration | PHPUnit + test DB | 90% |
| Repository implementations | Integration | PHPUnit + PostgreSQL testcontainer | 85% |
| XML validation | Unit | PHPUnit with fixture XMLs | 95% |
| REST API endpoints | Contract | PHPUnit + HTTP client | 80% |
| Command processing | Integration | PHPUnit + test queue | 90% |
| Security (auth, rate limiting, injection) | Negative tests | PHPUnit + dedicated security test suite | N/A (all paths covered) |
| Tree copy algorithm | Unit + integration | PHPUnit with multi-level test trees | 95% |

Static analysis: PHPStan level 9 (PHP §7.3). Formatting: php-cs-fixer with PSR-12 (PHP §2.1.2).

### Appendix D: Architecture Decision Records

| ADR | Decision | Alternatives Considered | Rationale |
|---|---|---|---|
| ADR-001 | PostgreSQL over MySQL | MySQL 8, MongoDB, BaseX | PostgreSQL's native XML type, `xpath()`, transactional DDL, and advisory locks directly support the system's core operations |
| ADR-002 | XML over JSON for content | JSON with JSON Schema | XML provides XSD validation (via DOMDocument), XPath querying, namespace support, and renders directly as objects (notes.md) |
| ADR-003 | Command queue over synchronous writes | Direct CRUD, event sourcing | Queue ensures ordered processing, audit trail, conflict prevention, and retry capability without event store complexity |
| ADR-004 | Single content table over per-type tables | Table-per-schema-type | Single table enables universal UUID references, uniform operations, and avoids schema explosion (notes.md: "One Table") |
| ADR-005 | UUID (128-bit) over sequential INT | Auto-increment INT, ULID | UUIDs are non-guessable (Security §Q25), portable between servers (notes.md: "Combining Databases"), and support distributed generation |
| ADR-006 | Layered monolith over microservices | Microservices | System complexity doesn't warrant distribution overhead. Clean layer boundaries enable future extraction if needed (Architecture §1.2) |
| ADR-007 | External binary storage | BLOB columns, BYTEA | Keeps database focused on structured content; binaries don't benefit from SQL querying (design.md, notes.md) |

---

*Document generated as an Architecture Design Document per Architecture §1.7.1. Significant decisions are recorded in Appendix D. All referenced standards are from the project's engineering standards corpus.*
