# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Graphic Design Tool · Created: 2026-05-20

## Philosophy

This approach stores core structural relationships (users, teams, projects, files, pages) as normalised relational tables, but represents the design content itself — shapes, their properties, fills, strokes, text content — as rich JSONB documents within a few broad tables. Each page stores its entire shape tree as a single JSONB column, and brand kits, templates, and component definitions use JSONB for their variable-structure content.

This is the pattern used by many modern SaaS platforms that need rapid iteration and flexible schemas. Penpot's architecture hints at this: shapes have a core set of geometric attributes but widely varying properties depending on type (text shapes have font data, path shapes have SVG d attributes, image shapes have asset references). Rather than creating 10+ sub-tables for each property type, a single JSONB column captures all properties with PostgreSQL's JSONB indexing providing query access.

The Hybrid approach is particularly well-suited to a graphic design tool because design files are fundamentally document-like: they are loaded as a whole, edited in-memory on the client, and saved back. Cross-file queries ("find all shapes with this color") are secondary to the primary access pattern of "load this file's content." JSONB excels at this document-load pattern while relational tables handle multi-tenant isolation, access control, and cross-file metadata queries.

**Best for:** Rapid MVP development, teams that want fewer migrations, and products where the primary access pattern is "load entire design file" rather than "query across all designs."

**Trade-offs:**
- (+) Far fewer tables (~15) — simpler schema, fewer migrations
- (+) Flexible shape properties — new shape types require no schema changes
- (+) Fast whole-file loading — one SELECT loads the entire page content
- (+) JSONB operators and GIN indexes enable rich queries when needed
- (+) Natural fit for client-side editing (load JSON, edit in-memory, save JSON back)
- (-) Cross-file queries (e.g., "all shapes with brand color X") require JSONB containment queries, which are slower than relational joins
- (-) Partial updates require JSONB merge operations (jsonb_set / || operator)
- (-) Schema validation must be enforced at application level, not database level
- (-) Large JSONB documents (1000+ shapes) can have performance implications for partial updates
- (-) Harder to write ad-hoc SQL reports against design content

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C SVG 2 | Shape objects in JSONB use SVG-aligned property names (x, y, width, height, d, transform) |
| ISO 32000-2 (PDF 2.0) | Export settings stored as JSONB with PDF standard and color space fields |
| ISO 15076-1 (ICC Profiles) | Brand kit JSONB includes color_space and cmyk_values for print-ready workflows |
| W3C WCAG 2.2 | Accessibility check results embedded as JSONB arrays with WCAG level and issue details |
| W3C DTCG Design Tokens | Design tokens extracted from brand kit JSONB for standards-compliant export |
| OpenType / WOFF2 | Font metadata in JSONB, binary font files in storage_objects |

---

## Users, Teams & Access Control

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
    preferences     JSONB DEFAULT '{}',     -- UI preferences, locale, timezone
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB DEFAULT '{}',
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
    -- File-level metadata as JSONB (canvas settings, grid, guides, etc.)
    settings        JSONB DEFAULT '{}',
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_files_project ON files (project_id);

CREATE TABLE file_library_links (
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    library_file_id UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    PRIMARY KEY (file_id, library_file_id)
);
```

## Pages — The Core Document Store

```sql
-- Each page stores its ENTIRE shape tree as a JSONB document
CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Page 1',
    sort_order      INTEGER NOT NULL DEFAULT 0,

    -- The full shape tree for this page
    content         JSONB NOT NULL DEFAULT '{"shapes": {}, "shape_order": []}',

    -- Page-level settings
    viewport        JSONB DEFAULT '{"x": 0, "y": 0, "zoom": 1.0}',
    grid_settings   JSONB DEFAULT '{}',
    guides          JSONB DEFAULT '[]',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_file ON pages (file_id);

-- GIN index for searching within page content
CREATE INDEX idx_pages_content ON pages USING GIN (content jsonb_path_ops);

-- The content JSONB structure:
-- {
--   "shapes": {
--     "uuid-1": {
--       "id": "uuid-1",
--       "type": "frame",
--       "name": "Instagram Post",
--       "parent_id": null,
--       "children": ["uuid-2", "uuid-3"],
--       "x": 0, "y": 0, "width": 1080, "height": 1080,
--       "rotation": 0,
--       "opacity": 1.0,
--       "visible": true,
--       "locked": false,
--       "blend_mode": "normal",
--       "constraints": {"horizontal": "left", "vertical": "top"},
--       "fills": [
--         {"type": "solid", "color": "#FFFFFF", "opacity": 1.0}
--       ],
--       "strokes": [],
--       "effects": []
--     },
--     "uuid-2": {
--       "id": "uuid-2",
--       "type": "text",
--       "name": "Headline",
--       "parent_id": "uuid-1",
--       "children": [],
--       "x": 50, "y": 200, "width": 980, "height": 100,
--       "text": {
--         "content": "Hello World",
--         "font_family": "Inter",
--         "font_size": 48,
--         "font_weight": 700,
--         "line_height": 1.2,
--         "text_align": "center",
--         "color": "#1A1A2E"
--       },
--       "fills": [],
--       "strokes": []
--     },
--     "uuid-3": {
--       "id": "uuid-3",
--       "type": "rect",
--       "name": "Background",
--       "parent_id": "uuid-1",
--       "children": [],
--       "x": 0, "y": 0, "width": 1080, "height": 1080,
--       "fills": [
--         {"type": "linear-gradient", "stops": [
--           {"offset": 0, "color": "#667eea"},
--           {"offset": 1, "color": "#764ba2"}
--         ]}
--       ],
--       "strokes": [],
--       "corner_radius": [0, 0, 0, 0]
--     }
--   },
--   "shape_order": ["uuid-1"]  -- root-level shape ordering
-- }
```

## Components

```sql
CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    component_set_name VARCHAR(255),         -- variant group name

    -- Component definition as JSONB (the shape subtree)
    definition      JSONB NOT NULL,
    -- {
    --   "root_shape": { ...shape object... },
    --   "properties": [
    --     {"name": "variant", "type": "enum", "values": ["primary", "secondary"]},
    --     {"name": "size", "type": "enum", "values": ["sm", "md", "lg"]},
    --     {"name": "label", "type": "text", "default": "Button"}
    --   ],
    --   "variants": {
    --     "primary/sm": { ...shape overrides... },
    --     "primary/md": { ...shape overrides... }
    --   }
    -- }

    thumbnail_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_components_file ON components (file_id);
```

## Brand Kit

```sql
CREATE TABLE brand_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Brand Kit',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,

    -- Entire brand kit content as JSONB
    colors          JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"id": "uuid", "name": "Primary", "hex": "#FF5733", "color_space": "srgb"},
    --   {"id": "uuid", "name": "Secondary", "hex": "#1A1A2E", "color_space": "srgb"},
    --   {"id": "uuid", "name": "Print Primary", "hex": "#FF5733", "color_space": "cmyk",
    --    "cmyk": [0, 64, 76, 0]}
    -- ]

    typography      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"id": "uuid", "name": "Heading", "font_id": "uuid", "font_family": "Inter",
    --    "size": 24, "weight": 700, "line_height": 1.2},
    --   {"id": "uuid", "name": "Body", "font_id": "uuid", "font_family": "Inter",
    --    "size": 16, "weight": 400, "line_height": 1.5}
    -- ]

    logos           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"id": "uuid", "name": "Primary Logo", "variant": "primary", "asset_id": "uuid"},
    --   {"id": "uuid", "name": "Icon Only", "variant": "icon", "asset_id": "uuid"}
    -- ]

    guidelines      JSONB DEFAULT '{}',
    -- {
    --   "min_logo_clearance_px": 24,
    --   "forbidden_colors": ["#FF0000"],
    --   "tone_of_voice": "professional, approachable"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_brand_kits_team ON brand_kits (team_id);
```

## Asset Storage

```sql
CREATE TABLE storage_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_path    TEXT NOT NULL,
    dimensions      JSONB,                   -- {"width": 1920, "height": 1080}
    metadata        JSONB DEFAULT '{}',      -- EXIF, ICC profile, etc.
    tags            TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_storage_objects_team ON storage_objects (team_id);
CREATE INDEX idx_storage_objects_tags ON storage_objects USING GIN (tags);

CREATE TABLE fonts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    family          VARCHAR(255) NOT NULL,
    variants        JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"style": "normal", "weight": 400, "format": "woff2", "asset_id": "uuid"},
    --   {"style": "italic", "weight": 400, "format": "woff2", "asset_id": "uuid"},
    --   {"style": "normal", "weight": 700, "format": "woff2", "asset_id": "uuid"}
    -- ]
    is_variable     BOOLEAN NOT NULL DEFAULT FALSE,
    variable_axes   JSONB,                   -- {"wght": {"min": 100, "max": 900}, "wdth": {...}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fonts_family ON fonts (family);
CREATE INDEX idx_fonts_team ON fonts (team_id);
```

## Templates

```sql
CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    subcategory     VARCHAR(100),
    dimensions      JSONB NOT NULL,          -- {"width": 1080, "height": 1080, "unit": "px"}
    thumbnail_url   TEXT,
    source_file_id  UUID NOT NULL REFERENCES files(id),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    tags            TEXT[],

    -- Autofill dataset definition (Canva-style)
    dataset         JSONB DEFAULT '{}',
    -- {
    --   "headline": {"type": "text", "max_length": 50},
    --   "hero_image": {"type": "image", "aspect_ratio": "16:9"},
    --   "cta_text": {"type": "text", "max_length": 20}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON templates (category, subcategory);
CREATE INDEX idx_templates_tags ON templates USING GIN (tags);
```

## Collaboration & Versioning

```sql
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id),
    content         TEXT NOT NULL,
    position        JSONB,                   -- {"x": 150, "y": 300, "shape_id": "uuid"}
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_file ON comments (file_id);

CREATE TABLE file_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    label           VARCHAR(255),
    -- Stores a snapshot of ALL pages' content JSONB
    pages_snapshot  JSONB NOT NULL,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (file_id, version_number)
);

CREATE INDEX idx_file_versions_file ON file_versions (file_id);
```

## AI Generation & Export

```sql
CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    job_type        VARCHAR(50) NOT NULL,
    prompt          TEXT NOT NULL,
    parameters      JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    result          JSONB,                   -- generated shapes, images, copy
    target_file_id  UUID REFERENCES files(id),
    tokens_used     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs (status);

CREATE TABLE export_presets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    -- {
    --   "format": "pdf",
    --   "color_space": "cmyk",
    --   "icc_profile_id": "uuid",
    --   "pdf_standard": "pdf-x-4",
    --   "include_bleed": true,
    --   "bleed_mm": 3.0,
    --   "quality": 90,
    --   "scale": 2.0
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Load entire page content (primary access pattern)

```sql
SELECT id, name, content, viewport, grid_settings, guides
FROM pages
WHERE file_id = '{{file_id}}'
ORDER BY sort_order;
-- Returns full shape tree as JSONB; client deserialises and renders
```

### Find shapes by type within a page

```sql
SELECT shape_key, shape_value
FROM pages p,
     jsonb_each(p.content->'shapes') AS shapes(shape_key, shape_value)
WHERE p.id = '{{page_id}}'
  AND shape_value->>'type' = 'text';
```

### Find all pages containing a specific brand color

```sql
SELECT p.id, p.name, f.name AS file_name
FROM pages p
JOIN files f ON f.id = p.file_id
JOIN projects proj ON proj.id = f.project_id
WHERE proj.team_id = '{{team_id}}'
  AND p.content @> '{"shapes": {}}' -- has shapes
  AND EXISTS (
      SELECT 1
      FROM jsonb_each(p.content->'shapes') AS s(k, v),
           jsonb_array_elements(v->'fills') AS fill
      WHERE fill->>'color' = '#FF5733'
  );
```

### Update a single shape property (partial JSONB update)

```sql
UPDATE pages
SET content = jsonb_set(
    content,
    ARRAY['shapes', '{{shape_id}}', 'x'],
    '150'::jsonb
),
    updated_at = now()
WHERE id = '{{page_id}}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 2 | users, team_memberships |
| Teams | 1 | teams |
| Projects & Files | 3 | projects, files, file_library_links |
| Design Content | 2 | pages (JSONB content), components (JSONB definition) |
| Brand Kit | 1 | brand_kits (all JSONB columns) |
| Assets & Storage | 2 | storage_objects, fonts |
| Templates | 1 | templates |
| Collaboration | 2 | comments, file_versions |
| AI & Export | 2 | ai_generation_jobs, export_presets |
| **Total** | **16** | |

---

## Key Design Decisions

1. **Page content as a single JSONB document** — the entire shape tree for a page lives in `pages.content`. This matches the primary access pattern: load a page, render all shapes, edit in-memory, save back. It eliminates the need for 6+ shape sub-tables and recursive CTEs. The trade-off is that cross-page queries require JSONB operators.

2. **Shapes stored as a flat map, not nested** — within the JSONB, shapes are stored as `{"shapes": {"uuid": {...}}}` with `parent_id` references, not as a nested tree. This makes partial updates (jsonb_set on a single shape) efficient and avoids deep nesting performance issues.

3. **GIN index on page content** — `jsonb_path_ops` GIN index enables containment queries (`@>`) for searching within design content. This is slower than relational queries but sufficient for brand compliance and asset usage reporting.

4. **Brand kit as JSONB columns** — colors, typography, and logos are JSONB arrays within a single `brand_kits` row. This keeps the schema simple and makes whole-kit loading fast. The trade-off is that "which designs use brand color X?" requires JSONB queries.

5. **Component definitions as JSONB** — component variants, properties, and shape overrides are stored as a rich JSONB document. This naturally handles the variable-structure nature of component systems (buttons have different properties than cards).

6. **Font variants as JSONB** — rather than one row per font weight/style, all variants are stored as a JSONB array. This matches how fonts are consumed (load all variants for a family).

7. **Template autofill dataset** — following Canva's pattern, templates define their fillable fields as a JSONB `dataset` schema. This enables programmatic template population without knowing the internal shape structure.

8. **Version snapshots store full page content** — `file_versions.pages_snapshot` captures the entire file state. Combined with the JSONB content model, this is straightforward: serialize all pages' content into one document. The trade-off is storage size for files with many versions.
