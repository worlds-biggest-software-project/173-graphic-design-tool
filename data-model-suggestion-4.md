# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Graphic Design Tool · Created: 2026-05-20

## Philosophy

This approach uses a graph layer to model the rich, interconnected relationships that emerge in a design platform — shape hierarchies, component instances referencing source components, brand kit usage across files, library dependencies between files, and collaboration relationships — while retaining relational tables for structured operational data (users, teams, billing).

A graphic design platform is fundamentally a graph problem: shapes form trees within pages, components create cross-file references, brand kits create usage relationships across every design, and files link to shared libraries. In a normalized relational model, these relationships require recursive CTEs, multi-table joins, and denormalized caches. In a graph model, traversals like "find all designs that use components from this library" or "which brand colors are used in which files" become natural path queries.

This model uses PostgreSQL with `ltree` for shape hierarchies and a lightweight property-graph pattern (`graph_nodes` / `graph_edges` tables) for cross-entity relationships. This avoids the operational overhead of a separate graph database (Neo4j) while providing graph query capabilities through recursive CTEs and ltree operators. For teams that need deeper graph capabilities, the same schema can front a dedicated graph database.

**Best for:** Platforms with complex cross-file relationships, design system management, brand compliance auditing, and impact analysis ("if I change this component, what breaks?").

**Trade-offs:**
- (+) Natural modeling of hierarchies, references, and dependencies
- (+) Impact analysis queries are straightforward ("what uses this component/color/font?")
- (+) ltree provides indexed hierarchical queries without recursive CTEs
- (+) Graph edges model any relationship type without schema changes
- (+) Design system management (component usage, library dependencies) is a first-class concern
- (-) Graph query patterns are less familiar to most developers
- (-) Property graph tables add abstraction overhead
- (-) ltree paths must be maintained when shapes are reparented
- (-) Two mental models (relational + graph) increase cognitive load
- (-) Aggregate queries (COUNT, SUM) across graph edges are less efficient than relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C SVG 2 | Shape nodes in the graph carry SVG-aligned properties; ltree paths mirror SVG DOM structure |
| ISO 32000-2 (PDF 2.0) | Export configuration nodes linked to files via graph edges |
| ISO 15076-1 (ICC Profiles) | Color profile nodes connected to brand kits and export presets via edges |
| W3C WCAG 2.2 | Accessibility violations modeled as edges between shapes and WCAG rules |
| W3C DTCG Design Tokens | Design tokens extracted by traversing brand kit graph relationships |
| OpenType / WOFF2 | Font nodes connected to brand typography and shape text via edges |

---

## Core Relational Tables

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

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

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_team ON projects (team_id);

CREATE TABLE files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    is_shared_library BOOLEAN NOT NULL DEFAULT FALSE,
    thumbnail_url   TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_files_project ON files (project_id);

CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Page 1',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    viewport        JSONB DEFAULT '{"x": 0, "y": 0, "zoom": 1.0}',
    grid_settings   JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_file ON pages (file_id);
```

## Shapes with ltree Hierarchy

```sql
-- Shapes use ltree for hierarchical path-based queries
CREATE TABLE shapes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES shapes(id) ON DELETE CASCADE,
    shape_type      VARCHAR(50) NOT NULL,
    name            VARCHAR(255),
    sort_order      INTEGER NOT NULL DEFAULT 0,

    -- ltree path: page_uuid.frame_uuid.group_uuid.shape_uuid
    -- Enables ancestor/descendant queries without recursive CTEs
    tree_path       LTREE NOT NULL,

    -- Geometric attributes (SVG-aligned)
    x               DOUBLE PRECISION DEFAULT 0,
    y               DOUBLE PRECISION DEFAULT 0,
    width           DOUBLE PRECISION DEFAULT 0,
    height          DOUBLE PRECISION DEFAULT 0,
    rotation        DOUBLE PRECISION DEFAULT 0,
    transform       DOUBLE PRECISION[6],

    -- Visual attributes
    opacity         DOUBLE PRECISION DEFAULT 1.0,
    visible         BOOLEAN NOT NULL DEFAULT TRUE,
    locked          BOOLEAN NOT NULL DEFAULT FALSE,
    blend_mode      VARCHAR(30) DEFAULT 'normal',

    -- All fills, strokes, effects, text, path data as JSONB
    -- (hybrid approach: relational for structure, JSONB for properties)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "fills": [{"type": "solid", "color": "#FF5733", "opacity": 1.0}],
    --   "strokes": [{"color": "#000000", "width": 1, "alignment": "center"}],
    --   "effects": [{"type": "drop-shadow", "x": 2, "y": 2, "blur": 4, "color": "#00000040"}],
    --   "constraints": {"horizontal": "left", "vertical": "top"},
    --   "corner_radius": [8, 8, 8, 8],
    --   -- Text-specific:
    --   "text": {"content": "Hello", "font_family": "Inter", "font_size": 16, ...},
    --   -- Path-specific:
    --   "path_data": "M 0 0 L 100 100 ...",
    --   "fill_rule": "nonzero"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shapes_page ON shapes (page_id);
CREATE INDEX idx_shapes_parent ON shapes (parent_id);
CREATE INDEX idx_shapes_type ON shapes (shape_type);
CREATE INDEX idx_shapes_tree_path ON shapes USING GIST (tree_path);
CREATE INDEX idx_shapes_properties ON shapes USING GIN (properties jsonb_path_ops);
```

## Property Graph Layer

```sql
-- Generic graph nodes for entities that participate in cross-entity relationships
-- Some nodes reference relational tables (shapes, files, components); others are graph-only
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: 'component', 'component_instance', 'brand_color', 'brand_font',
    --             'brand_logo', 'brand_kit', 'font', 'asset', 'template',
    --             'color_value', 'ai_generation'
    entity_id       UUID,                    -- FK to relational table (shape, file, etc.)
    entity_table    VARCHAR(50),             -- 'shapes', 'files', 'storage_objects', etc.
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes (entity_table, entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties jsonb_path_ops);

-- Typed, directed edges between graph nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(100) NOT NULL,
    -- Edge types:
    --   'instance_of'        — component instance -> source component
    --   'uses_color'         — shape -> brand color
    --   'uses_font'          — shape -> font
    --   'uses_asset'         — shape -> storage object (image fill)
    --   'library_of'         — library file -> consuming file
    --   'derived_from'       — AI-generated shape -> generation job
    --   'variant_of'         — component variant -> component set
    --   'exported_as'        — file -> export (format, color space)
    --   'violates'           — shape -> accessibility rule
    --   'belongs_to_kit'     — color/font/logo -> brand kit
    properties      JSONB DEFAULT '{}',      -- edge metadata (overrides, weights, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_source_type ON graph_edges (source_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges (target_id, edge_type);
```

## Brand Kit (Graph-Native)

```sql
-- Brand kits are modeled as graph subgraphs
-- A brand_kit node connects to color, font, and logo nodes via edges

CREATE TABLE brand_kits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Brand Kit',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- graph node for this kit
    guidelines      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_brand_kits_team ON brand_kits (team_id);

-- Brand colors, fonts, and logos are graph_nodes with 'belongs_to_kit' edges
-- to the brand_kit node. Their properties are stored in graph_nodes.properties:
--
-- Brand color node properties:
-- {"name": "Primary", "hex": "#FF5733", "color_space": "srgb", "cmyk": [0,64,76,0]}
--
-- Brand font node properties:
-- {"name": "Heading", "font_family": "Inter", "font_size": 24, "weight": 700}
--
-- Brand logo node properties:
-- {"name": "Primary Logo", "variant": "primary", "asset_id": "uuid"}
```

## Components (Graph-Connected)

```sql
CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    root_shape_id   UUID NOT NULL REFERENCES shapes(id),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),  -- graph node for this component
    variant_properties JSONB DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_components_file ON components (file_id);

-- Component instances are tracked via graph edges:
-- instance shape node --[instance_of]--> component node
-- This enables "find all instances of this component across all files"
```

## Assets, Templates, Collaboration

```sql
CREATE TABLE storage_objects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_path    TEXT NOT NULL,
    dimensions      JSONB,
    metadata        JSONB DEFAULT '{}',
    node_id         UUID REFERENCES graph_nodes(id),  -- graph node for relationship tracking
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_storage_objects_team ON storage_objects (team_id);

CREATE TABLE fonts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    family          VARCHAR(255) NOT NULL,
    variants        JSONB NOT NULL DEFAULT '[]',
    is_variable     BOOLEAN NOT NULL DEFAULT FALSE,
    node_id         UUID REFERENCES graph_nodes(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    subcategory     VARCHAR(100),
    dimensions      JSONB NOT NULL,
    source_file_id  UUID NOT NULL REFERENCES files(id),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    tags            TEXT[],
    dataset         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id),
    content         TEXT NOT NULL,
    position        JSONB,
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE file_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    label           VARCHAR(255),
    snapshot_data   JSONB NOT NULL,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (file_id, version_number)
);

CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    job_type        VARCHAR(50) NOT NULL,
    prompt          TEXT NOT NULL,
    parameters      JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    result_data     JSONB,
    node_id         UUID REFERENCES graph_nodes(id),  -- graph node for provenance tracking
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

---

## Example Queries

### All descendants of a frame (ltree, no recursive CTE)

```sql
-- Find all shapes inside a frame using ltree
SELECT id, name, shape_type, tree_path
FROM shapes
WHERE tree_path <@ (
    SELECT tree_path FROM shapes WHERE id = '{{frame_id}}'
)
ORDER BY tree_path;
```

### All siblings of a shape

```sql
SELECT id, name, shape_type
FROM shapes
WHERE parent_id = (
    SELECT parent_id FROM shapes WHERE id = '{{shape_id}}'
)
  AND id != '{{shape_id}}'
ORDER BY sort_order;
```

### Impact analysis: find all instances of a component across all files

```sql
SELECT
    s.id AS instance_shape_id,
    s.name AS instance_name,
    f.name AS file_name,
    p.name AS page_name
FROM graph_edges ge
JOIN graph_nodes source_node ON source_node.id = ge.source_id
JOIN shapes s ON s.id = source_node.entity_id AND source_node.entity_table = 'shapes'
JOIN pages p ON p.id = s.page_id
JOIN files f ON f.id = p.file_id
WHERE ge.edge_type = 'instance_of'
  AND ge.target_id = (
      SELECT node_id FROM components WHERE id = '{{component_id}}'
  );
```

### Brand compliance: find shapes using a non-brand color

```sql
WITH brand_colors AS (
    SELECT gn.properties->>'hex' AS hex
    FROM graph_edges ge
    JOIN graph_nodes gn ON gn.id = ge.source_id
    WHERE ge.edge_type = 'belongs_to_kit'
      AND ge.target_id = (SELECT node_id FROM brand_kits WHERE team_id = '{{team_id}}' AND is_default = TRUE)
      AND gn.node_type = 'brand_color'
)
SELECT s.id, s.name, fill->>'color' AS used_color
FROM shapes s
JOIN pages p ON p.id = s.page_id
JOIN files f ON f.id = p.file_id
JOIN projects proj ON proj.id = f.project_id,
     jsonb_array_elements(s.properties->'fills') AS fill
WHERE proj.team_id = '{{team_id}}'
  AND fill->>'type' = 'solid'
  AND fill->>'color' NOT IN (SELECT hex FROM brand_colors);
```

### Dependency graph: which files depend on a shared library?

```sql
SELECT
    f.id AS consuming_file_id,
    f.name AS consuming_file_name,
    proj.name AS project_name
FROM graph_edges ge
JOIN graph_nodes source_node ON source_node.id = ge.source_id
JOIN graph_nodes target_node ON target_node.id = ge.target_id
JOIN files f ON f.id = source_node.entity_id AND source_node.entity_table = 'files'
JOIN projects proj ON proj.id = f.project_id
WHERE ge.edge_type = 'library_of'
  AND target_node.entity_id = '{{library_file_id}}'
  AND target_node.entity_table = 'files';
```

### AI provenance: trace which shapes were AI-generated

```sql
SELECT
    s.id AS shape_id,
    s.name AS shape_name,
    aj.prompt,
    aj.job_type,
    aj.created_at AS generated_at
FROM graph_edges ge
JOIN graph_nodes shape_node ON shape_node.id = ge.source_id
JOIN graph_nodes ai_node ON ai_node.id = ge.target_id
JOIN shapes s ON s.id = shape_node.entity_id
JOIN ai_generation_jobs aj ON aj.node_id = ai_node.id
WHERE ge.edge_type = 'derived_from'
  AND shape_node.entity_table = 'shapes';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 2 | users, team_memberships |
| Teams | 1 | teams |
| Projects & Files | 2 | projects, files |
| Design Content | 2 | pages, shapes (with ltree) |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Components | 1 | components (graph-connected) |
| Brand Kit | 1 | brand_kits (graph-native) |
| Assets & Storage | 2 | storage_objects, fonts |
| Templates | 1 | templates |
| Collaboration | 2 | comments, file_versions |
| AI Generation | 1 | ai_generation_jobs |
| **Total** | **17** | Plus graph_nodes/graph_edges for extensible relationships |

---

## Key Design Decisions

1. **ltree for shape hierarchies** — PostgreSQL's `ltree` extension provides indexed hierarchical queries (ancestor, descendant, depth, path matching) without recursive CTEs. The `tree_path` column stores the full path from page root to shape (e.g., `page_uuid.frame_uuid.group_uuid.shape_uuid`). Trade-off: paths must be updated when shapes are reparented, but this is an infrequent operation.

2. **Property graph on PostgreSQL** — `graph_nodes` and `graph_edges` implement a lightweight property graph within PostgreSQL. This avoids the operational complexity of a separate graph database while providing relationship-first query patterns. For teams that outgrow this pattern, the same schema can be mirrored to Neo4j or Amazon Neptune.

3. **Hybrid shape properties** — geometric attributes (x, y, width, height) are relational columns for direct querying and indexing; variable properties (fills, strokes, effects, text content) are in a JSONB `properties` column. This balances query performance for spatial queries with schema flexibility for diverse shape types.

4. **Graph-native brand management** — brand colors, fonts, and logos are graph nodes connected to a brand kit node via edges. This makes "which designs use this brand color?" a natural graph traversal rather than a complex cross-table join. Edge properties can carry usage metadata (e.g., "used as primary background color").

5. **Component instances as graph edges** — rather than a separate `component_instances` table, the instance-of relationship is modeled as a graph edge (`instance_of`) from the instance shape node to the component node. This makes impact analysis ("what breaks if I change this component?") a single-hop graph query.

6. **AI provenance tracking** — AI-generated shapes are connected to their generation jobs via `derived_from` edges. This enables provenance queries ("was this image AI-generated?") and fine-tuning analytics ("which prompts produce the best results?").

7. **Extensible edge types** — new relationship types can be added without schema changes. If the product adds features like "design approvals" or "asset licensing," new edge types (`approved_by`, `licensed_from`) are simply new string values in `graph_edges.edge_type`.

8. **Dual indexing strategy** — shapes are indexed both by ltree (for hierarchical queries within a page) and via graph edges (for cross-file relationship queries). This serves two different query patterns: rendering a single page vs. analyzing relationships across the entire workspace.
