# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Graphic Design Tool · Created: 2026-05-20

## Philosophy

This approach treats the design file as an append-only event log rather than mutable relational state. Every user action — adding a shape, changing a color, moving a layer, applying a brand style — is captured as an immutable event in a single `design_events` table. The current state of any design is always derived by replaying events from the beginning (or from a periodic snapshot). Materialised views and read models provide fast query access to the current state.

This is the architecture used by collaborative editing systems built on Operational Transform (OT) or CRDTs, and is the foundation of undo/redo, version history, and real-time collaboration in tools like Figma. Event sourcing naturally provides a complete audit trail (who changed what, when, and why), temporal queries ("what did this design look like last Tuesday?"), and conflict-free multiplayer editing when combined with CRDTs.

The trade-off is increased complexity: write paths are simple (append an event), but read paths require materialisation. Snapshot compaction is essential to prevent event replay from becoming slow as files grow large.

**Best for:** Platforms where full audit trails, real-time collaboration, temporal queries, and undo/redo are core requirements — and where the team is comfortable with CQRS patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail of every design change
- (+) Temporal queries: reconstruct design state at any point in time
- (+) Natural foundation for undo/redo and version history
- (+) Real-time collaboration via event broadcasting (WebSocket + event stream)
- (+) AI analytics on change patterns (most-edited elements, design iteration velocity)
- (-) Read path requires materialised views or snapshot replay
- (-) Snapshot compaction needed to prevent slow replays on large files
- (-) More complex query patterns (CQRS) than simple SELECT
- (-) Storage grows with every edit, not just current state
- (-) Debugging requires understanding event replay semantics

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C SVG 2 | Event payloads reference SVG node types and geometric attributes; materialised shapes map to SVG elements |
| ISO 32000-2 (PDF 2.0) | Export events capture PDF generation parameters and color space selections |
| ISO 15076-1 (ICC Profiles) | Color profile assignments captured as events; current profile derived from latest event |
| W3C WCAG 2.2 | Accessibility check events store WCAG level and violation details |
| W3C DTCG Design Tokens | Token export events reference materialised brand kit state |
| OCSF-inspired event schema | Event envelope structure (actor, action, target, timestamp) follows structured event log patterns |
| RFC 7519 (JWT) | User identity in events derived from authenticated JWT claims |

---

## Core Event Store

```sql
-- The single source of truth for all design changes
CREATE TABLE design_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate ID (file_id for design events)
    stream_type     VARCHAR(50) NOT NULL,   -- 'design_file', 'brand_kit', 'template'
    sequence_num    BIGINT NOT NULL,         -- monotonically increasing per stream
    event_type      VARCHAR(100) NOT NULL,   -- e.g., 'shape.created', 'shape.moved', 'fill.changed'
    event_data      JSONB NOT NULL,          -- event-specific payload
    actor_id        UUID NOT NULL,           -- user who performed the action
    actor_ip        INET,
    session_id      UUID,                    -- collaboration session
    correlation_id  UUID,                    -- groups related events (e.g., multi-shape move)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
);

-- Optimised for stream replay (most common read pattern)
CREATE INDEX idx_events_stream_seq ON design_events (stream_id, sequence_num);
-- Optimised for global event feed (webhooks, analytics)
CREATE INDEX idx_events_created ON design_events (created_at);
-- Optimised for user activity queries
CREATE INDEX idx_events_actor ON design_events (actor_id, created_at);
-- Optimised for event type filtering
CREATE INDEX idx_events_type ON design_events (event_type);

-- Event type taxonomy:
--   shape.created, shape.deleted, shape.moved, shape.resized, shape.rotated
--   shape.fill_changed, shape.stroke_changed, shape.opacity_changed
--   shape.text_changed, shape.path_changed, shape.renamed
--   shape.locked, shape.unlocked, shape.visibility_changed
--   shape.reordered, shape.reparented
--   page.created, page.deleted, page.renamed, page.reordered
--   component.created, component.detached, component.override_changed
--   brand.color_added, brand.color_changed, brand.font_added
--   ai.generation_requested, ai.generation_completed
--   comment.added, comment.resolved
--   export.requested, export.completed
--   collaboration.user_joined, collaboration.user_left

-- Example event_data for 'shape.created':
-- {
--   "shape_id": "uuid",
--   "parent_id": "uuid",
--   "shape_type": "rect",
--   "name": "Background",
--   "x": 0, "y": 0, "width": 1080, "height": 1080,
--   "fills": [{"type": "solid", "color": "#FFFFFF"}],
--   "sort_order": 0
-- }

-- Example event_data for 'shape.moved':
-- {
--   "shape_id": "uuid",
--   "from": {"x": 100, "y": 200},
--   "to": {"x": 150, "y": 250}
-- }
```

## Snapshots (Compaction)

```sql
-- Periodic snapshots to avoid full replay on every load
CREATE TABLE design_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    sequence_num    BIGINT NOT NULL,         -- snapshot taken after this event
    snapshot_data   JSONB NOT NULL,          -- full materialised state at this point
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_snapshots_stream ON design_snapshots (stream_id, sequence_num DESC);

-- Snapshot is taken every N events (e.g., every 100 or 500 events per stream)
-- To load current state:
--   1. Find latest snapshot for stream
--   2. Replay events after snapshot's sequence_num
--   3. Apply events to snapshot state = current state
```

## Materialised Read Models

These tables are projections of the event stream, rebuilt from events. They are the read side of the CQRS pattern.

```sql
-- Materialised current state of all shapes (rebuilt from events)
CREATE TABLE shapes_view (
    id              UUID PRIMARY KEY,
    page_id         UUID NOT NULL,
    file_id         UUID NOT NULL,
    parent_id       UUID,
    shape_type      VARCHAR(50) NOT NULL,
    name            VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    x               DOUBLE PRECISION DEFAULT 0,
    y               DOUBLE PRECISION DEFAULT 0,
    width           DOUBLE PRECISION DEFAULT 0,
    height          DOUBLE PRECISION DEFAULT 0,
    rotation        DOUBLE PRECISION DEFAULT 0,
    opacity         DOUBLE PRECISION DEFAULT 1.0,
    visible         BOOLEAN NOT NULL DEFAULT TRUE,
    locked          BOOLEAN NOT NULL DEFAULT FALSE,
    blend_mode      VARCHAR(30) DEFAULT 'normal',
    fills           JSONB DEFAULT '[]',
    strokes         JSONB DEFAULT '[]',
    text_content    JSONB,                   -- for text shapes
    path_data       TEXT,                    -- SVG path d attribute
    constraints     JSONB DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,         -- last event that updated this shape
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shapes_view_page ON shapes_view (page_id);
CREATE INDEX idx_shapes_view_file ON shapes_view (file_id);
CREATE INDEX idx_shapes_view_parent ON shapes_view (parent_id);

-- Materialised current state of pages
CREATE TABLE pages_view (
    id              UUID PRIMARY KEY,
    file_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    viewport        JSONB DEFAULT '{}',
    grid_settings   JSONB DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_view_file ON pages_view (file_id);
```

## Organisational Tables (Not Event-Sourced)

User management, teams, and infrastructure tables use standard relational models since they change infrequently and don't benefit from event sourcing.

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,
    auth_provider   VARCHAR(50),
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id)
);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_shared_library BOOLEAN NOT NULL DEFAULT FALSE,
    current_seq     BIGINT NOT NULL DEFAULT 0,  -- latest event sequence number
    snapshot_seq    BIGINT NOT NULL DEFAULT 0,   -- latest snapshot sequence number
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE storage_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_path    TEXT NOT NULL,
    width           INTEGER,
    height          INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Brand Kit (Event-Sourced)

```sql
-- Brand kit changes are event-sourced for audit trail
-- Events go into design_events with stream_type = 'brand_kit'

-- Materialised brand kit view
CREATE TABLE brand_kits_view (
    id              UUID PRIMARY KEY,
    team_id         UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    colors          JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id": "uuid", "name": "Primary", "hex": "#FF5733", "cmyk": [0,64,76,0]}]
    typography      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id": "uuid", "name": "Heading", "family": "Inter", "size": 24, "weight": 700}]
    logos           JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id": "uuid", "name": "Primary Logo", "variant": "primary", "asset_id": "uuid"}]
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_brand_kits_view_team ON brand_kits_view (team_id);
```

## Collaboration Sessions

```sql
-- Active collaboration sessions (ephemeral, not event-sourced)
CREATE TABLE collaboration_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    cursor_x        DOUBLE PRECISION,
    cursor_y        DOUBLE PRECISION,
    page_id         UUID,
    selection        UUID[],                 -- currently selected shape IDs
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_heartbeat  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collab_sessions_file ON collaboration_sessions (file_id);
```

## Templates & AI Generation

```sql
CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    width           DOUBLE PRECISION NOT NULL,
    height          DOUBLE PRECISION NOT NULL,
    source_file_id  UUID NOT NULL REFERENCES files(id),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    tags            TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI generation is event-sourced: ai.generation_requested -> ai.generation_completed
-- This means we have a full history of all AI generations, prompts, and results
-- Materialised view for active/recent jobs:
CREATE TABLE ai_jobs_view (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    team_id         UUID NOT NULL,
    job_type        VARCHAR(50) NOT NULL,
    prompt          TEXT NOT NULL,
    parameters      JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL,
    result_data     JSONB,
    tokens_used     INTEGER DEFAULT 0,
    requested_at    TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ
);
```

---

## Example Queries

### Reconstruct design state at a specific point in time

```sql
-- Get the latest snapshot before the target time
WITH target_snapshot AS (
    SELECT snapshot_data, sequence_num
    FROM design_snapshots
    WHERE stream_id = '{{file_id}}'
      AND created_at <= '2026-05-15 14:00:00+00'
    ORDER BY sequence_num DESC
    LIMIT 1
),
-- Get events between snapshot and target time
subsequent_events AS (
    SELECT event_type, event_data, sequence_num
    FROM design_events
    WHERE stream_id = '{{file_id}}'
      AND sequence_num > (SELECT sequence_num FROM target_snapshot)
      AND created_at <= '2026-05-15 14:00:00+00'
    ORDER BY sequence_num
)
SELECT * FROM target_snapshot
UNION ALL
SELECT NULL, to_jsonb(e.*), NULL FROM subsequent_events e;
-- Application layer replays events onto snapshot to reconstruct state
```

### Audit trail: who changed a specific shape?

```sql
SELECT
    e.event_type,
    e.event_data,
    e.created_at,
    u.display_name AS actor_name
FROM design_events e
JOIN users u ON u.id = e.actor_id
WHERE e.stream_id = '{{file_id}}'
  AND e.event_data->>'shape_id' = '{{shape_id}}'
ORDER BY e.sequence_num;
```

### Design iteration analytics

```sql
-- How many edits per hour (design velocity)
SELECT
    date_trunc('hour', created_at) AS hour,
    COUNT(*) AS edit_count,
    COUNT(DISTINCT actor_id) AS active_users
FROM design_events
WHERE stream_id = '{{file_id}}'
  AND created_at >= now() - INTERVAL '7 days'
GROUP BY 1
ORDER BY 1;
```

### Undo: get the last N events for a user session

```sql
SELECT id, event_type, event_data, sequence_num
FROM design_events
WHERE stream_id = '{{file_id}}'
  AND actor_id = '{{user_id}}'
  AND session_id = '{{session_id}}'
ORDER BY sequence_num DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | design_events, design_snapshots |
| Materialised Read Models | 4 | shapes_view, pages_view, brand_kits_view, ai_jobs_view |
| Users & Auth | 2 | users, team_memberships |
| Teams & Organizations | 1 | teams |
| Projects & Files | 2 | projects, files |
| Assets & Storage | 1 | storage_objects |
| Collaboration | 1 | collaboration_sessions |
| Templates | 1 | templates |
| **Total** | **14** | Plus any additional read model projections as needed |

---

## Key Design Decisions

1. **Single event table for all design changes** — all events go into `design_events` regardless of type, using `stream_id` and `stream_type` for partitioning. This simplifies the write path and enables global event feeds. PostgreSQL table partitioning by `created_at` should be added for production scale.

2. **Event-type taxonomy with dotted notation** — events use `entity.action` naming (e.g., `shape.created`, `shape.moved`) making it easy to filter by entity or action type. New shape properties or entity types can be added by defining new event types without schema changes.

3. **Snapshots for read performance** — without snapshots, loading a design file that has been edited 10,000 times would require replaying 10,000 events. Snapshots every 100-500 events bound replay cost. The snapshot cadence is tunable per file based on edit frequency.

4. **Materialised views for the read path** — `shapes_view`, `pages_view`, and `brand_kits_view` are projections maintained by an event processor. They provide fast reads but are derived data — the event log is always the source of truth. These views can be rebuilt from the event log at any time.

5. **Organisational data is NOT event-sourced** — users, teams, and projects use standard relational tables. Event sourcing adds complexity that is not justified for entities that change infrequently and don't need temporal queries.

6. **Brand kit changes ARE event-sourced** — brand kits benefit from audit trails ("who changed the brand color?") and temporal queries ("what was our brand typography last quarter?"), so they share the event store.

7. **Correlation IDs group related events** — when a user moves 5 shapes at once, all 5 `shape.moved` events share a `correlation_id`. This enables grouped undo and meaningful audit trail display.

8. **Collaboration sessions are ephemeral** — cursor positions and selections are stored in a `collaboration_sessions` table (or Redis) and are not event-sourced, as they have no long-term value and would bloat the event store.
