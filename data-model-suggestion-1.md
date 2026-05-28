# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Graphic Design Tool · Created: 2026-05-20

## Philosophy

This approach models every concept in the graphic design platform as a discrete, strongly-typed relational table with enforced foreign key relationships. Every entity — from teams and projects to individual shapes on a canvas — has its own table with explicit columns, constraints, and indexes. The design file's node tree is flattened into a `shapes` table using an adjacency list pattern, where each shape references its parent via `parent_id`.

This mirrors how traditional design tools (Penpot, early Figma) store their data: a clear hierarchy of Profile -> Team -> Project -> File -> Page -> Shape, with separate tables for assets, templates, brand kits, fonts, and collaboration artefacts. The normalized approach maximises data integrity and makes cross-entity queries (e.g., "find all files using this brand color") straightforward SQL joins.

Real-world systems that use this pattern include Penpot (PostgreSQL with normalized tables for profiles, teams, projects, files, and shapes) and traditional DAM platforms. It is the most predictable architecture for a team that values schema clarity and strong referential integrity.

**Best for:** Teams prioritising data integrity, complex cross-entity queries, and a clear, auditable schema that maps directly to domain concepts.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Straightforward SQL queries for reporting and analytics
- (+) Easy to reason about — every concept has a table
- (+) Standard PostgreSQL tooling (migrations, backups, monitoring) works out of the box
- (-) High table count (~35-40 tables) increases migration complexity
- (-) Schema changes for new shape types or properties require ALTER TABLE migrations
- (-) Adjacency list for node tree requires recursive CTEs for deep traversal
- (-) RBAC permission checks involve multi-table joins

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C SVG 2 | Shape types map to SVG node types (rect, ellipse, path, text, group, frame); geometric attributes (x, y, width, height) stored as columns |
| ISO 32000-2 (PDF 2.0) | Export metadata (bleed, crop marks, color space) stored in `export_presets` table |
| ISO 15076-1 (ICC Profiles) | Color profiles stored in `color_profiles` table with binary ICC data in `storage_objects` |
| W3C WCAG 2.2 | Accessibility check results stored in `accessibility_reports` table per design |
| W3C DTCG Design Tokens | Design tokens exported from `brand_colors`, `brand_typography`, `brand_spacing` tables |
| OpenType / WOFF2 | Font metadata in `fonts` table; binary font files in `storage_objects` |
| OAuth 2.0 / JWT | Authentication tokens reference `users` and `team_memberships` tables |

---

## Entity Management

### Users & Authentication

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,              -- NULL if SSO-only
    auth_provider   VARCHAR(50),       -- 'local', 'google', 'saml'
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    locale          VARCHAR(10) DEFAULT 'en',
    timezone        VARCHAR(50) DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_auth_provider ON users (auth_provider, auth_provider_id);
```

### Teams & Organizations

```sql
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- 'free', 'pro', 'enterprise'
    logo_url        TEXT,
    settings        JSONB DEFAULT '{}',  -- team-level preferences
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'owner', 'admin', 'member', 'viewer'
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id)
);

CREATE INDEX idx_team_memberships_team ON team_memberships (team_id);
CREATE INDEX idx_team_memberships_user ON team_memberships (user_id);
```

## Projects & Files

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_team ON projects (team_id);

CREATE TABLE files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_shared_library BOOLEAN NOT NULL DEFAULT FALSE,
    thumbnail_url   TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_files_project ON files (project_id);
CREATE INDEX idx_files_shared_library ON files (is_shared_library) WHERE is_shared_library = TRUE;

-- Files may use other files as shared libraries (Penpot pattern)
CREATE TABLE file_library_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    library_file_id UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (file_id, library_file_id)
);
```

## Pages & Shapes (Design Node Tree)

```sql
CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Page 1',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    viewport_x      DOUBLE PRECISION DEFAULT 0,
    viewport_y      DOUBLE PRECISION DEFAULT 0,
    viewport_zoom   DOUBLE PRECISION DEFAULT 1.0,
    grid_settings   JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_file ON pages (file_id);

-- The core entity: each shape = one SVG node / design layer
CREATE TABLE shapes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES shapes(id) ON DELETE CASCADE,  -- adjacency list
    shape_type      VARCHAR(50) NOT NULL,  -- 'frame', 'group', 'rect', 'ellipse', 'path', 'text', 'image', 'component', 'instance'
    name            VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,

    -- Geometric attributes (SVG-aligned)
    x               DOUBLE PRECISION DEFAULT 0,
    y               DOUBLE PRECISION DEFAULT 0,
    width           DOUBLE PRECISION DEFAULT 0,
    height          DOUBLE PRECISION DEFAULT 0,
    rotation        DOUBLE PRECISION DEFAULT 0,

    -- Transform matrix (for complex transforms)
    transform       DOUBLE PRECISION[6],  -- [a, b, c, d, e, f] SVG transform matrix

    -- Visual attributes
    opacity         DOUBLE PRECISION DEFAULT 1.0,
    visible         BOOLEAN NOT NULL DEFAULT TRUE,
    locked          BOOLEAN NOT NULL DEFAULT FALSE,
    blend_mode      VARCHAR(30) DEFAULT 'normal',

    -- Constraints (for responsive resizing)
    constraint_h    VARCHAR(30) DEFAULT 'left',    -- 'left', 'right', 'center', 'stretch', 'scale'
    constraint_v    VARCHAR(30) DEFAULT 'top',     -- 'top', 'bottom', 'center', 'stretch', 'scale'

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shapes_page ON shapes (page_id);
CREATE INDEX idx_shapes_parent ON shapes (parent_id);
CREATE INDEX idx_shapes_type ON shapes (shape_type);

-- Fills and strokes (many per shape)
CREATE TABLE shape_fills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shape_id        UUID NOT NULL REFERENCES shapes(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    fill_type       VARCHAR(30) NOT NULL,  -- 'solid', 'linear-gradient', 'radial-gradient', 'image'
    color           VARCHAR(9),            -- hex color e.g. '#FF5733FF'
    opacity         DOUBLE PRECISION DEFAULT 1.0,
    gradient_stops  JSONB,                 -- [{offset: 0, color: '#FFF'}, ...]
    image_asset_id  UUID REFERENCES storage_objects(id)
);

CREATE INDEX idx_shape_fills_shape ON shape_fills (shape_id);

CREATE TABLE shape_strokes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shape_id        UUID NOT NULL REFERENCES shapes(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    stroke_type     VARCHAR(30) NOT NULL DEFAULT 'solid',
    color           VARCHAR(9),
    width           DOUBLE PRECISION DEFAULT 1.0,
    alignment       VARCHAR(20) DEFAULT 'center',  -- 'center', 'inside', 'outside'
    dash_pattern    DOUBLE PRECISION[]
);

CREATE INDEX idx_shape_strokes_shape ON shape_strokes (shape_id);

-- Text-specific properties
CREATE TABLE shape_text_content (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shape_id        UUID NOT NULL REFERENCES shapes(id) ON DELETE CASCADE,
    content         TEXT NOT NULL,
    font_family     VARCHAR(255),
    font_size       DOUBLE PRECISION DEFAULT 16,
    font_weight     INTEGER DEFAULT 400,
    font_style      VARCHAR(20) DEFAULT 'normal',
    line_height     DOUBLE PRECISION DEFAULT 1.5,
    letter_spacing  DOUBLE PRECISION DEFAULT 0,
    text_align      VARCHAR(20) DEFAULT 'left',
    text_decoration VARCHAR(30) DEFAULT 'none',
    color           VARCHAR(9)
);

CREATE INDEX idx_shape_text_shape ON shape_text_content (shape_id);

-- Path data for vector shapes
CREATE TABLE shape_path_data (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shape_id        UUID NOT NULL REFERENCES shapes(id) ON DELETE CASCADE,
    svg_path        TEXT NOT NULL,          -- SVG path d attribute
    fill_rule       VARCHAR(20) DEFAULT 'nonzero'  -- 'nonzero', 'evenodd'
);
```

## Components & Design Systems

```sql
CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    root_shape_id   UUID NOT NULL REFERENCES shapes(id),
    component_set_id UUID REFERENCES component_sets(id),  -- for variant groups
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE component_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Component instances reference their source component
CREATE TABLE component_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shape_id        UUID NOT NULL REFERENCES shapes(id) ON DELETE CASCADE,
    component_id    UUID NOT NULL REFERENCES components(id),
    overrides       JSONB DEFAULT '{}',  -- property overrides from the source component
    is_detached     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_component_instances_component ON component_instances (component_id);
```

## Brand Kit

```sql
CREATE TABLE brand_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Brand Kit',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE brand_colors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_kit_id    UUID NOT NULL REFERENCES brand_kits(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    hex_value       VARCHAR(9) NOT NULL,   -- '#FF5733FF'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    color_space     VARCHAR(20) DEFAULT 'srgb',  -- 'srgb', 'display-p3', 'cmyk' (ISO 15076-1)
    cmyk_values     DOUBLE PRECISION[4]    -- for print workflows
);

CREATE TABLE brand_typography (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_kit_id    UUID NOT NULL REFERENCES brand_kits(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,  -- 'Heading', 'Body', 'Caption'
    font_id         UUID REFERENCES fonts(id),
    font_size       DOUBLE PRECISION NOT NULL,
    font_weight     INTEGER DEFAULT 400,
    line_height     DOUBLE PRECISION DEFAULT 1.5,
    letter_spacing  DOUBLE PRECISION DEFAULT 0,
    sort_order      INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE brand_logos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_kit_id    UUID NOT NULL REFERENCES brand_kits(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    variant         VARCHAR(50),            -- 'primary', 'secondary', 'icon', 'monochrome'
    asset_id        UUID NOT NULL REFERENCES storage_objects(id),
    sort_order      INTEGER NOT NULL DEFAULT 0
);
```

## Asset Storage & Templates

```sql
CREATE TABLE storage_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,   -- MIME type
    size_bytes      BIGINT NOT NULL,
    storage_backend VARCHAR(50) NOT NULL DEFAULT 's3',  -- 's3', 'gcs', 'local'
    storage_path    TEXT NOT NULL,
    width           INTEGER,                 -- for images
    height          INTEGER,                 -- for images
    metadata        JSONB DEFAULT '{}',      -- EXIF, ICC profile info, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_storage_objects_team ON storage_objects (team_id);

CREATE TABLE fonts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),  -- NULL = system font
    family          VARCHAR(255) NOT NULL,
    style           VARCHAR(50) NOT NULL DEFAULT 'normal',
    weight          INTEGER NOT NULL DEFAULT 400,
    format          VARCHAR(20) NOT NULL,    -- 'otf', 'ttf', 'woff2' (OpenType/WOFF2)
    asset_id        UUID REFERENCES storage_objects(id),
    is_variable     BOOLEAN NOT NULL DEFAULT FALSE,  -- OpenType Variable Font
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fonts_family ON fonts (family);
CREATE INDEX idx_fonts_team ON fonts (team_id);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),  -- NULL = system template
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),           -- 'social-media', 'presentation', 'infographic', 'ad', 'print'
    subcategory     VARCHAR(100),           -- 'instagram-post', 'facebook-cover', 'a4-flyer'
    width           DOUBLE PRECISION NOT NULL,
    height          DOUBLE PRECISION NOT NULL,
    unit            VARCHAR(10) DEFAULT 'px',  -- 'px', 'mm', 'in'
    thumbnail_url   TEXT,
    source_file_id  UUID NOT NULL REFERENCES files(id),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    tags            TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON templates (category, subcategory);
CREATE INDEX idx_templates_tags ON templates USING GIN (tags);
CREATE INDEX idx_templates_team ON templates (team_id);
```

## Collaboration

```sql
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,  -- threaded replies
    content         TEXT NOT NULL,
    x               DOUBLE PRECISION,       -- pin position on canvas
    y               DOUBLE PRECISION,
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_file ON comments (file_id);

CREATE TABLE file_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    label           VARCHAR(255),
    snapshot_data   JSONB NOT NULL,          -- serialized file state
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (file_id, version_number)
);

CREATE INDEX idx_file_versions_file ON file_versions (file_id);
```

## Export & Accessibility

```sql
CREATE TABLE export_presets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    format          VARCHAR(20) NOT NULL,    -- 'png', 'jpg', 'svg', 'pdf', 'webp', 'avif'
    scale           DOUBLE PRECISION DEFAULT 1.0,
    quality         INTEGER DEFAULT 90,
    color_space     VARCHAR(20) DEFAULT 'srgb',  -- 'srgb', 'cmyk' (ISO 15076-1)
    icc_profile_id  UUID REFERENCES storage_objects(id),
    pdf_standard    VARCHAR(20),             -- 'pdf-x-1a', 'pdf-x-4' (ISO 32000-2)
    include_bleed   BOOLEAN DEFAULT FALSE,
    bleed_mm        DOUBLE PRECISION DEFAULT 3.0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE accessibility_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id),
    generated_by    UUID NOT NULL REFERENCES users(id),
    wcag_level      VARCHAR(5) NOT NULL DEFAULT 'AA',  -- 'A', 'AA', 'AAA' (WCAG 2.2)
    issues          JSONB NOT NULL DEFAULT '[]',
    passed          BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## AI Generation

```sql
CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    job_type        VARCHAR(50) NOT NULL,    -- 'text-to-image', 'text-to-layout', 'copy-generation', 'variant-generation'
    prompt          TEXT NOT NULL,
    parameters      JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- 'pending', 'processing', 'completed', 'failed'
    result_data     JSONB,
    target_file_id  UUID REFERENCES files(id),
    target_page_id  UUID REFERENCES pages(id),
    tokens_used     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_ai_jobs_user ON ai_generation_jobs (user_id);
CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs (status);
```

---

## Example Queries

### Recursive tree traversal (all descendants of a frame)

```sql
WITH RECURSIVE shape_tree AS (
    SELECT id, name, shape_type, parent_id, 0 AS depth
    FROM shapes
    WHERE id = '{{frame_id}}'

    UNION ALL

    SELECT s.id, s.name, s.shape_type, s.parent_id, st.depth + 1
    FROM shapes s
    JOIN shape_tree st ON s.parent_id = st.id
)
SELECT * FROM shape_tree ORDER BY depth, sort_order;
```

### Find all files using a specific brand color

```sql
SELECT DISTINCT f.id, f.name
FROM files f
JOIN pages p ON p.file_id = f.id
JOIN shapes s ON s.page_id = p.id
JOIN shape_fills sf ON sf.shape_id = s.id
WHERE sf.color = '#FF5733FF'
  AND f.project_id IN (
      SELECT id FROM projects WHERE team_id = '{{team_id}}'
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 2 | users, team_memberships |
| Teams & Organizations | 1 | teams |
| Projects & Files | 4 | projects, files, file_library_links, file_versions |
| Pages & Shapes | 6 | pages, shapes, shape_fills, shape_strokes, shape_text_content, shape_path_data |
| Components | 3 | components, component_sets, component_instances |
| Brand Kit | 4 | brand_kits, brand_colors, brand_typography, brand_logos |
| Assets & Storage | 3 | storage_objects, fonts, templates |
| Collaboration | 1 | comments |
| Export & Accessibility | 2 | export_presets, accessibility_reports |
| AI Generation | 1 | ai_generation_jobs |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Adjacency list for the shape tree** — each shape has a `parent_id` referencing its parent shape. This is the simplest model for tree structures and matches how SVG nodes are nested. Recursive CTEs handle traversal. Alternative approaches (nested sets, materialized paths) add complexity without clear benefit given typical design file depths (rarely > 20 levels).

2. **Separate tables for fills, strokes, and text content** — rather than embedding these as JSONB on the shape, separate tables enforce type safety and enable direct querying (e.g., "find all shapes with this color"). The trade-off is more joins when loading a shape.

3. **Brand kit as normalized tables** — colors, typography, and logos each get their own table rather than a JSONB blob. This enables queries like "which designs use this brand color" and enforces consistency.

4. **Storage objects as a single table** — all binary assets (images, fonts, ICC profiles) share one table with a `storage_backend` discriminator. This simplifies asset management and enables a single CDN pipeline.

5. **File versions store full snapshots** — `file_versions.snapshot_data` contains a serialized copy of the entire file state (all pages and shapes). This is simple but storage-intensive. Event-sourced alternatives (Suggestion 2) trade storage for complexity.

6. **Templates reference source files** — a template is a metadata wrapper around a design file, not a separate data structure. This avoids duplicating the entire shape tree schema for templates.

7. **AI generation jobs as a queue table** — generation requests are tracked as stateful jobs with prompt, parameters, and result data. This supports async processing and usage tracking for billing.

8. **Team-scoped multi-tenancy with shared tables** — all tenant data is isolated by `team_id` foreign keys rather than separate schemas. Row-level security (RLS) policies should be added for enforcement at the database level.
