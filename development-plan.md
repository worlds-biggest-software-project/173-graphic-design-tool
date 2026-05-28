# Graphic Design Tool — Phased Development Plan

> Project: 173-graphic-design-tool · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (full-stack) | Canvas rendering, WebSocket collaboration, and rich client-side interactions demand a unified JS/TS stack; the entire competitor landscape (Figma, Canva, Penpot) uses TypeScript for both client and server |
| API framework | Fastify (Node.js) | High-throughput WebSocket support needed for real-time collaboration; native TypeScript; OpenAPI schema generation via `@fastify/swagger`; outperforms Express by 2-3x on benchmarks |
| Frontend framework | React 19 + Zustand | React dominates design-tool UIs (Figma, Excalidraw); Zustand provides lightweight state management suited to canvas editors where Redux overhead is unnecessary |
| Canvas rendering | HTML5 Canvas (2D) + custom render engine | SVG DOM becomes a bottleneck above ~500 nodes; HTML5 Canvas with a virtual scene graph (Figma's approach) scales to thousands of shapes; fallback SVG export for accessibility |
| Database | PostgreSQL 16 | JSONB for flexible shape properties, GIN indexes for content search, Row-Level Security for multi-tenancy; chosen Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary approach for its balance of schema simplicity and query capability |
| Cache / real-time | Redis 7 | Collaboration session state (cursor positions, selections), pub/sub for WebSocket event fanout, AI job queue, rate limiting |
| Task queue | BullMQ (Redis-backed) | TypeScript-native, Redis-backed job queue for async AI generation, export rendering, thumbnail generation; supports retries, priorities, and rate limiting |
| Object storage | S3-compatible (MinIO for self-hosted) | Binary asset storage (images, fonts, ICC profiles, exported files); MinIO for local dev and self-hosted; AWS S3 / GCS for SaaS |
| AI integration | OpenAI API (GPT-4o + DALL-E 3) + Anthropic API | Text-to-layout via structured output (GPT-4o), image generation via DALL-E 3 API, copy generation via Claude; provider-agnostic adapter layer for swappability |
| Authentication | Lucia Auth + OAuth 2.0 | Lightweight TypeScript auth library; supports local password + Google/GitHub/SAML SSO; session-based with JWT for API tokens; aligns with RFC 6749/7519 |
| Testing | Vitest + Playwright + Supertest | Vitest for unit/integration (fast, ESM-native); Playwright for E2E browser tests (canvas interaction); Supertest for API integration tests |
| Code quality | ESLint + Prettier + TypeScript strict mode | Standard TypeScript toolchain; strict mode catches type errors at compile time |
| Package manager | pnpm | Efficient monorepo support, disk-space savings via content-addressable store |
| Containerisation | Docker + Docker Compose | Required for self-hosted deployment; multi-service orchestration (API, worker, Redis, PostgreSQL, MinIO) |
| Monorepo | pnpm workspaces | Shared types between client and server; single lockfile; shared ESLint/Prettier config |

### Project Structure

```
graphic-design-tool/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── .env.example
├── packages/
│   ├── shared/                     # Shared types, constants, utilities
│   │   ├── src/
│   │   │   ├── types/
│   │   │   │   ├── shapes.ts       # Shape type definitions
│   │   │   │   ├── brand.ts        # Brand kit types
│   │   │   │   ├── collaboration.ts
│   │   │   │   ├── ai.ts           # AI generation types
│   │   │   │   └── export.ts       # Export/format types
│   │   │   ├── constants/
│   │   │   │   ├── formats.ts      # Export format presets (100+ sizes)
│   │   │   │   └── defaults.ts     # Default shape/style values
│   │   │   └── utils/
│   │   │       ├── color.ts        # Color space conversions (sRGB, CMYK, P3)
│   │   │       ├── geometry.ts     # Transform matrix, bounding box utilities
│   │   │       └── validation.ts   # Zod schemas for shared types
│   │   └── package.json
│   ├── api/                        # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts           # Fastify app setup, plugin registration
│   │   │   ├── routes/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── teams.ts
│   │   │   │   ├── projects.ts
│   │   │   │   ├── files.ts
│   │   │   │   ├── pages.ts
│   │   │   │   ├── templates.ts
│   │   │   │   ├── brand-kits.ts
│   │   │   │   ├── assets.ts
│   │   │   │   ├── ai.ts
│   │   │   │   ├── export.ts
│   │   │   │   └── comments.ts
│   │   │   ├── services/
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── file.service.ts
│   │   │   │   ├── page.service.ts
│   │   │   │   ├── brand.service.ts
│   │   │   │   ├── template.service.ts
│   │   │   │   ├── ai-generation.service.ts
│   │   │   │   ├── export.service.ts
│   │   │   │   ├── accessibility.service.ts
│   │   │   │   └── storage.service.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── rate-limit.ts
│   │   │   │   └── team-scope.ts
│   │   │   ├── ws/
│   │   │   │   ├── collaboration.ts   # WebSocket handler for real-time editing
│   │   │   │   └── presence.ts        # Cursor/selection presence
│   │   │   ├── db/
│   │   │   │   ├── client.ts          # PostgreSQL connection (pg + Kysely)
│   │   │   │   ├── migrations/
│   │   │   │   └── seeds/
│   │   │   └── plugins/
│   │   │       ├── swagger.ts
│   │   │       └── cors.ts
│   │   ├── tests/
│   │   └── package.json
│   ├── worker/                     # BullMQ background workers
│   │   ├── src/
│   │   │   ├── workers/
│   │   │   │   ├── ai-generation.worker.ts
│   │   │   │   ├── export.worker.ts
│   │   │   │   ├── thumbnail.worker.ts
│   │   │   │   └── accessibility-check.worker.ts
│   │   │   └── index.ts
│   │   ├── tests/
│   │   └── package.json
│   └── client/                     # React frontend
│       ├── src/
│       │   ├── main.tsx
│       │   ├── App.tsx
│       │   ├── stores/
│       │   │   ├── editor.store.ts     # Canvas state (shapes, selection, viewport)
│       │   │   ├── file.store.ts       # File/page management
│       │   │   ├── collaboration.store.ts
│       │   │   ├── brand.store.ts
│       │   │   └── ai.store.ts
│       │   ├── engine/
│       │   │   ├── renderer.ts         # Canvas 2D render loop
│       │   │   ├── scene-graph.ts      # Virtual scene graph
│       │   │   ├── hit-testing.ts      # Shape selection / click detection
│       │   │   ├── transform.ts        # Shape manipulation (move, resize, rotate)
│       │   │   ├── text-engine.ts      # Text measurement and rendering
│       │   │   └── viewport.ts         # Pan/zoom/scroll handling
│       │   ├── components/
│       │   │   ├── canvas/
│       │   │   │   ├── Canvas.tsx
│       │   │   │   ├── ShapeRenderer.tsx
│       │   │   │   └── SelectionOverlay.tsx
│       │   │   ├── panels/
│       │   │   │   ├── LayersPanel.tsx
│       │   │   │   ├── PropertiesPanel.tsx
│       │   │   │   ├── TemplatePanel.tsx
│       │   │   │   ├── BrandPanel.tsx
│       │   │   │   └── AIPanel.tsx
│       │   │   ├── toolbar/
│       │   │   │   ├── Toolbar.tsx
│       │   │   │   └── ShapeTools.tsx
│       │   │   └── modals/
│       │   │       ├── ExportModal.tsx
│       │   │       └── TemplatePickerModal.tsx
│       │   ├── hooks/
│       │   │   ├── useCanvas.ts
│       │   │   ├── useCollaboration.ts
│       │   │   ├── useUndoRedo.ts
│       │   │   └── useKeyboardShortcuts.ts
│       │   ├── api/
│       │   │   └── client.ts           # API client (fetch wrapper)
│       │   └── styles/
│       ├── tests/
│       ├── index.html
│       ├── vite.config.ts
│       └── package.json
├── migrations/
│   └── 001_initial.sql
└── scripts/
    ├── seed-templates.ts
    └── seed-fonts.ts
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose
Establish the monorepo structure, database schema, authentication system, and core API skeleton. After this phase, a developer can register an account, create a team, and authenticate against the API. All subsequent phases build on this scaffold.

### Tasks

#### 1.1 — Monorepo & Toolchain Setup

**What**: Initialize the pnpm workspace with four packages (shared, api, worker, client), configure TypeScript, ESLint, Prettier, and Docker Compose.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
```

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "paths": {
      "@graphic-tool/shared": ["../shared/src"]
    }
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: graphic_tool
      POSTGRES_USER: graphic_tool
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Unit: pnpm install completes without errors across all workspaces`
- `Unit: TypeScript compilation succeeds in all four packages`
- `Unit: ESLint passes on initial codebase with zero warnings`
- `Integration: docker-compose up starts PostgreSQL, Redis, and MinIO; healthchecks pass`

---

#### 1.2 — Database Schema & Migrations

**What**: Create the initial PostgreSQL schema using Kysely migrations, implementing the Hybrid Relational + JSONB data model (Data Model Suggestion 3).

**Design**:

```typescript
// packages/api/src/db/client.ts
import { Pool } from 'pg';
import { Kysely, PostgresDialect } from 'kysely';
import type { Database } from '@graphic-tool/shared';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
});

export const db = new Kysely<Database>({
  dialect: new PostgresDialect({ pool }),
});
```

```typescript
// packages/shared/src/types/database.ts
export interface Database {
  users: UsersTable;
  teams: TeamsTable;
  team_memberships: TeamMembershipsTable;
  projects: ProjectsTable;
  files: FilesTable;
  file_library_links: FileLibraryLinksTable;
  pages: PagesTable;
  components: ComponentsTable;
  brand_kits: BrandKitsTable;
  storage_objects: StorageObjectsTable;
  fonts: FontsTable;
  templates: TemplatesTable;
  comments: CommentsTable;
  file_versions: FileVersionsTable;
  ai_generation_jobs: AIGenerationJobsTable;
  export_presets: ExportPresetsTable;
}

export interface UsersTable {
  id: string;                    // UUID
  email: string;
  display_name: string;
  avatar_url: string | null;
  password_hash: string | null;
  auth_provider: 'local' | 'google' | 'github' | 'saml' | null;
  auth_provider_id: string | null;
  email_verified: boolean;
  preferences: Record<string, unknown>;  // JSONB
  created_at: Date;
  updated_at: Date;
}

export interface TeamsTable {
  id: string;
  name: string;
  slug: string;
  plan: 'free' | 'pro' | 'enterprise';
  settings: Record<string, unknown>;
  created_at: Date;
  updated_at: Date;
}

export interface TeamMembershipsTable {
  id: string;
  team_id: string;
  user_id: string;
  role: 'owner' | 'admin' | 'member' | 'viewer';
  joined_at: Date;
}

export interface ProjectsTable {
  id: string;
  team_id: string;
  name: string;
  description: string | null;
  is_archived: boolean;
  created_by: string;
  created_at: Date;
  updated_at: Date;
}

export interface FilesTable {
  id: string;
  project_id: string;
  name: string;
  description: string | null;
  is_shared_library: boolean;
  thumbnail_url: string | null;
  version: number;
  settings: Record<string, unknown>;  // JSONB (canvas settings)
  created_by: string;
  created_at: Date;
  updated_at: Date;
}

export interface PagesTable {
  id: string;
  file_id: string;
  name: string;
  sort_order: number;
  content: PageContent;           // JSONB - entire shape tree
  viewport: { x: number; y: number; zoom: number };
  grid_settings: Record<string, unknown>;
  guides: unknown[];
  created_at: Date;
  updated_at: Date;
}

export interface PageContent {
  shapes: Record<string, ShapeNode>;
  shape_order: string[];          // root-level shape IDs in order
}
```

SQL migration (001_initial.sql):

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,
    auth_provider   VARCHAR(50),
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    preferences     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_memberships (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id)
);

CREATE INDEX idx_team_memberships_team ON team_memberships (team_id);
CREATE INDEX idx_team_memberships_user ON team_memberships (user_id);

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
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
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_shared_library BOOLEAN NOT NULL DEFAULT FALSE,
    thumbnail_url   TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
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

CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Page 1',
    sort_order      INTEGER NOT NULL DEFAULT 0,
    content         JSONB NOT NULL DEFAULT '{"shapes": {}, "shape_order": []}',
    viewport        JSONB DEFAULT '{"x": 0, "y": 0, "zoom": 1.0}',
    grid_settings   JSONB DEFAULT '{}',
    guides          JSONB DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pages_file ON pages (file_id);
CREATE INDEX idx_pages_content ON pages USING GIN (content jsonb_path_ops);

CREATE TABLE components (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    component_set_name VARCHAR(255),
    definition      JSONB NOT NULL,
    thumbnail_url   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_components_file ON components (file_id);

CREATE TABLE storage_objects (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID REFERENCES teams(id),
    uploader_id     UUID NOT NULL REFERENCES users(id),
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_path    TEXT NOT NULL,
    dimensions      JSONB,
    metadata        JSONB DEFAULT '{}',
    tags            TEXT[],
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_storage_objects_team ON storage_objects (team_id);
CREATE INDEX idx_storage_objects_tags ON storage_objects USING GIN (tags);

CREATE TABLE fonts (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID REFERENCES teams(id),
    family          VARCHAR(255) NOT NULL,
    variants        JSONB NOT NULL DEFAULT '[]',
    is_variable     BOOLEAN NOT NULL DEFAULT FALSE,
    variable_axes   JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fonts_family ON fonts (family);
CREATE INDEX idx_fonts_team ON fonts (team_id);

CREATE TABLE brand_kits (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL DEFAULT 'Brand Kit',
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    colors          JSONB NOT NULL DEFAULT '[]',
    typography      JSONB NOT NULL DEFAULT '[]',
    logos           JSONB NOT NULL DEFAULT '[]',
    guidelines      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_brand_kits_team ON brand_kits (team_id);

CREATE TABLE templates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    subcategory     VARCHAR(100),
    dimensions      JSONB NOT NULL,
    thumbnail_url   TEXT,
    source_file_id  UUID NOT NULL REFERENCES files(id),
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    tags            TEXT[],
    dataset         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_category ON templates (category, subcategory);
CREATE INDEX idx_templates_tags ON templates USING GIN (tags);

CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id),
    content         TEXT NOT NULL,
    position        JSONB,
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_file ON comments (file_id);

CREATE TABLE file_versions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_id         UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    label           VARCHAR(255),
    pages_snapshot  JSONB NOT NULL,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (file_id, version_number)
);

CREATE INDEX idx_file_versions_file ON file_versions (file_id);

CREATE TABLE ai_generation_jobs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID NOT NULL REFERENCES users(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    job_type        VARCHAR(50) NOT NULL,
    prompt          TEXT NOT NULL,
    parameters      JSONB DEFAULT '{}',
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    result          JSONB,
    target_file_id  UUID REFERENCES files(id),
    tokens_used     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_ai_jobs_status ON ai_generation_jobs (status);

CREATE TABLE export_presets (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    team_id         UUID REFERENCES teams(id),
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Testing**:
- `Unit: Kysely Database type matches all table interfaces without type errors`
- `Integration: migration 001 runs against fresh PostgreSQL; all tables created with correct indexes`
- `Integration: migration rollback drops all tables cleanly`
- `Integration: GIN index on pages.content enables JSONB containment queries`

---

#### 1.3 — Authentication & User Management

**What**: Implement user registration, login (email/password), OAuth 2.0 (Google, GitHub), session management, and JWT-based API tokens.

**Design**:

```typescript
// packages/api/src/routes/auth.ts
import { FastifyInstance } from 'fastify';

export async function authRoutes(app: FastifyInstance) {
  // POST /api/auth/register
  app.post<{ Body: RegisterBody }>('/api/auth/register', {
    schema: {
      body: RegisterBodySchema,
      response: { 201: AuthResponseSchema },
    },
  }, registerHandler);

  // POST /api/auth/login
  app.post<{ Body: LoginBody }>('/api/auth/login', {
    schema: {
      body: LoginBodySchema,
      response: { 200: AuthResponseSchema },
    },
  }, loginHandler);

  // GET /api/auth/oauth/:provider (Google, GitHub)
  app.get('/api/auth/oauth/:provider', oauthRedirectHandler);

  // GET /api/auth/oauth/:provider/callback
  app.get('/api/auth/oauth/:provider/callback', oauthCallbackHandler);

  // POST /api/auth/logout
  app.post('/api/auth/logout', { preHandler: [requireAuth] }, logoutHandler);

  // GET /api/auth/me
  app.get('/api/auth/me', { preHandler: [requireAuth] }, meHandler);
}

interface RegisterBody {
  email: string;
  password: string;
  display_name: string;
}

interface LoginBody {
  email: string;
  password: string;
}

interface AuthResponse {
  user: { id: string; email: string; display_name: string; avatar_url: string | null };
  session_token: string;
}
```

```typescript
// packages/api/src/services/auth.service.ts
import { hash, verify } from '@node-rs/argon2';

export class AuthService {
  async register(input: RegisterBody): Promise<AuthResponse>;
  async login(email: string, password: string): Promise<AuthResponse>;
  async createOAuthUser(provider: string, profile: OAuthProfile): Promise<AuthResponse>;
  async validateSession(token: string): Promise<User | null>;
  async createSession(userId: string): Promise<string>;
  async revokeSession(token: string): Promise<void>;
}
```

Password hashing uses Argon2id (recommended by OWASP). Sessions stored in PostgreSQL with Redis-cached lookups.

**Testing**:
- `Unit: register with valid email/password -> user created, session token returned`
- `Unit: register with duplicate email -> 409 Conflict`
- `Unit: register with weak password (<8 chars) -> 400 Bad Request`
- `Unit: login with correct credentials -> session token returned`
- `Unit: login with wrong password -> 401 Unauthorized`
- `Unit: login with nonexistent email -> 401 Unauthorized`
- `Integration (mocked OAuth): Google OAuth callback with valid code -> user created, session set`
- `Integration (mocked OAuth): OAuth callback with existing email -> accounts linked`
- `Unit: validateSession with valid token -> user object returned`
- `Unit: validateSession with expired token -> null returned`
- `Unit: logout -> session revoked, subsequent validateSession returns null`

---

#### 1.4 — Team & Project CRUD

**What**: Implement API endpoints for creating and managing teams, team memberships, and projects.

**Design**:

```typescript
// API endpoints
// POST   /api/teams                    -> create team
// GET    /api/teams                    -> list user's teams
// GET    /api/teams/:teamId            -> get team details
// PATCH  /api/teams/:teamId            -> update team settings
// POST   /api/teams/:teamId/members    -> invite member
// DELETE /api/teams/:teamId/members/:userId -> remove member
// PATCH  /api/teams/:teamId/members/:userId -> change role

// POST   /api/teams/:teamId/projects   -> create project
// GET    /api/teams/:teamId/projects   -> list projects
// GET    /api/projects/:projectId      -> get project details
// PATCH  /api/projects/:projectId      -> update project
// DELETE /api/projects/:projectId      -> archive project

interface CreateTeamBody {
  name: string;
}

interface CreateTeamResponse {
  id: string;
  name: string;
  slug: string;          // auto-generated from name, unique
  plan: 'free';
  created_at: string;
}

interface InviteMemberBody {
  email: string;
  role: 'admin' | 'member' | 'viewer';
}
```

**Testing**:
- `Unit: create team -> team created with auto-slug, creator is owner`
- `Unit: create team with duplicate slug -> slug appended with random suffix`
- `Unit: list teams -> returns only teams where user is a member`
- `Unit: invite member -> team_membership created with invited role`
- `Unit: invite member to team by non-admin -> 403 Forbidden`
- `Unit: remove last owner from team -> 400 Bad Request (must have at least one owner)`
- `Unit: create project -> project created under team, creator recorded`
- `Unit: list projects -> returns only projects for the specified team`
- `Unit: archive project -> is_archived set to true, project excluded from default listings`
- `Integration: team-scoped middleware rejects requests for teams user is not a member of`

---

## Phase 2: Canvas Engine & Shape System

### Purpose
Build the client-side canvas rendering engine that can display and manipulate shapes. After this phase, a user can create a design file, add shapes (rectangles, ellipses, text, images), move/resize/rotate them, and see the result rendered in real-time on an HTML5 Canvas. This is the technical core of the entire product.

### Tasks

#### 2.1 — Scene Graph & Shape Types

**What**: Define the shape type system and implement a virtual scene graph that manages the shape tree in memory.

**Design**:

```typescript
// packages/shared/src/types/shapes.ts

export type ShapeType =
  | 'frame'
  | 'group'
  | 'rect'
  | 'ellipse'
  | 'path'
  | 'text'
  | 'image';

export interface BaseShape {
  id: string;
  type: ShapeType;
  name: string;
  parent_id: string | null;
  children: string[];           // ordered child IDs
  x: number;
  y: number;
  width: number;
  height: number;
  rotation: number;             // degrees
  opacity: number;              // 0-1
  visible: boolean;
  locked: boolean;
  blend_mode: BlendMode;
  constraints: {
    horizontal: 'left' | 'right' | 'center' | 'stretch' | 'scale';
    vertical: 'top' | 'bottom' | 'center' | 'stretch' | 'scale';
  };
  fills: Fill[];
  strokes: Stroke[];
  effects: Effect[];
}

export interface Fill {
  type: 'solid' | 'linear-gradient' | 'radial-gradient' | 'image';
  color?: string;              // hex, e.g., '#FF5733FF'
  opacity?: number;
  gradient_stops?: { offset: number; color: string }[];
  image_asset_id?: string;
}

export interface Stroke {
  color: string;
  width: number;
  alignment: 'center' | 'inside' | 'outside';
  dash_pattern?: number[];
}

export interface Effect {
  type: 'drop-shadow' | 'inner-shadow' | 'blur';
  x?: number;
  y?: number;
  blur: number;
  spread?: number;
  color?: string;
}

export interface RectShape extends BaseShape {
  type: 'rect';
  corner_radius: [number, number, number, number]; // TL, TR, BR, BL
}

export interface EllipseShape extends BaseShape {
  type: 'ellipse';
  arc_start: number;           // degrees, 0-360
  arc_end: number;
  inner_radius: number;        // 0-1, for donut shapes
}

export interface TextShape extends BaseShape {
  type: 'text';
  text: TextContent;
}

export interface TextContent {
  content: string;
  font_family: string;
  font_size: number;
  font_weight: number;
  font_style: 'normal' | 'italic';
  line_height: number;
  letter_spacing: number;
  text_align: 'left' | 'center' | 'right' | 'justify';
  text_decoration: 'none' | 'underline' | 'strikethrough';
  color: string;
}

export interface PathShape extends BaseShape {
  type: 'path';
  path_data: string;           // SVG path d attribute
  fill_rule: 'nonzero' | 'evenodd';
}

export interface ImageShape extends BaseShape {
  type: 'image';
  asset_id: string;
  crop: { x: number; y: number; width: number; height: number } | null;
}

export interface FrameShape extends BaseShape {
  type: 'frame';
  clip_content: boolean;
  corner_radius: [number, number, number, number];
}

export interface GroupShape extends BaseShape {
  type: 'group';
}

export type ShapeNode =
  | RectShape
  | EllipseShape
  | TextShape
  | PathShape
  | ImageShape
  | FrameShape
  | GroupShape;
```

```typescript
// packages/client/src/engine/scene-graph.ts
export class SceneGraph {
  private shapes: Map<string, ShapeNode>;
  private rootOrder: string[];

  constructor(content: PageContent);

  getShape(id: string): ShapeNode | undefined;
  getChildren(parentId: string): ShapeNode[];
  getParent(shapeId: string): ShapeNode | null;
  getAncestors(shapeId: string): ShapeNode[];
  getDescendants(shapeId: string): ShapeNode[];

  addShape(shape: ShapeNode, parentId: string | null, index?: number): void;
  removeShape(shapeId: string): ShapeNode[];   // returns removed subtree
  moveShape(shapeId: string, newParentId: string | null, index: number): void;
  updateShape(shapeId: string, updates: Partial<ShapeNode>): void;
  reorderShape(shapeId: string, newIndex: number): void;

  toPageContent(): PageContent;
  getShapesInRenderOrder(): ShapeNode[];       // depth-first traversal
  getShapesAtPoint(x: number, y: number): ShapeNode[];  // hit testing
}
```

**Testing**:
- `Unit: SceneGraph constructor -> builds shape map from PageContent JSONB`
- `Unit: addShape to root -> shape appears in rootOrder at specified index`
- `Unit: addShape as child of frame -> shape appears in parent's children array`
- `Unit: removeShape with children -> entire subtree removed, IDs cleaned from parent`
- `Unit: moveShape between parents -> children arrays updated correctly`
- `Unit: getShapesInRenderOrder -> returns depth-first traversal respecting sort order`
- `Unit: getAncestors -> returns path from shape to root`
- `Unit: toPageContent -> round-trips correctly (construct -> modify -> serialize -> construct)`
- `Unit: getShapesAtPoint with overlapping shapes -> returns shapes in reverse render order (topmost first)`

---

#### 2.2 — Canvas Renderer

**What**: Implement the HTML5 Canvas 2D rendering engine that draws shapes from the scene graph.

**Design**:

```typescript
// packages/client/src/engine/renderer.ts
export class CanvasRenderer {
  private ctx: CanvasRenderingContext2D;
  private sceneGraph: SceneGraph;
  private viewport: Viewport;
  private dpr: number;          // device pixel ratio
  private frameId: number | null;

  constructor(canvas: HTMLCanvasElement, sceneGraph: SceneGraph);

  setViewport(viewport: Viewport): void;
  startRenderLoop(): void;
  stopRenderLoop(): void;
  requestRender(): void;        // mark dirty, render next frame

  // Core render pipeline (called each frame)
  private render(): void {
    // 1. Clear canvas
    // 2. Apply viewport transform (pan/zoom)
    // 3. Traverse scene graph in render order
    // 4. For each visible shape: save ctx -> apply transform -> draw fills ->
    //    draw strokes -> draw effects -> restore ctx
    // 5. Draw selection overlay (handled separately)
  }

  private drawShape(shape: ShapeNode): void;
  private drawRect(shape: RectShape): void;
  private drawEllipse(shape: EllipseShape): void;
  private drawText(shape: TextShape): void;
  private drawImage(shape: ImageShape): void;
  private drawPath(shape: PathShape): void;
  private drawFrame(shape: FrameShape): void;

  private applyFills(shape: BaseShape): void;
  private applyStrokes(shape: BaseShape): void;
  private applyEffects(shape: BaseShape): void;

  // Image loading/caching
  private imageCache: Map<string, HTMLImageElement>;
  loadImage(assetId: string, url: string): Promise<void>;
}

export interface Viewport {
  x: number;       // pan offset X
  y: number;       // pan offset Y
  zoom: number;    // 0.1 to 10.0
}
```

**Testing**:
- `Unit: render empty scene graph -> canvas cleared, no errors`
- `Unit: render single rect with solid fill -> correct rectangle drawn at position`
- `Unit: render rect with gradient fill -> linear gradient applied correctly`
- `Unit: render text shape -> text drawn with correct font, size, color, alignment`
- `Unit: render with viewport zoom 2x -> shapes appear at double size`
- `Unit: render with viewport pan (100, 200) -> shapes offset correctly`
- `Unit: render nested group -> children clipped to parent frame when clip_content is true`
- `Unit: render shape with opacity 0.5 -> globalAlpha applied`
- `Unit: render shape with visible=false -> shape not drawn`
- `Unit: render order -> shapes drawn in scene graph depth-first order (later shapes on top)`
- `E2E (Playwright): create canvas, add shapes via store, verify pixel colors at expected positions`

---

#### 2.3 — Shape Manipulation (Move, Resize, Rotate)

**What**: Implement mouse/touch interaction for selecting, moving, resizing, and rotating shapes on the canvas.

**Design**:

```typescript
// packages/client/src/engine/transform.ts
export class TransformManager {
  private sceneGraph: SceneGraph;
  private selection: SelectionState;

  constructor(sceneGraph: SceneGraph);

  // Hit testing: find shape under mouse position
  hitTest(canvasX: number, canvasY: number, viewport: Viewport): ShapeNode | null;

  // Selection
  select(shapeId: string): void;
  addToSelection(shapeId: string): void;
  deselectAll(): void;
  getSelection(): string[];

  // Transform operations
  moveShapes(shapeIds: string[], dx: number, dy: number): void;
  resizeShape(shapeId: string, handle: ResizeHandle, dx: number, dy: number,
              preserveAspect: boolean): void;
  rotateShape(shapeId: string, angleDelta: number): void;

  // Resize handles
  getResizeHandles(shapeId: string): HandlePosition[];
  getRotateHandle(shapeId: string): { x: number; y: number };
}

export type ResizeHandle = 'nw' | 'n' | 'ne' | 'e' | 'se' | 's' | 'sw' | 'w';

export interface HandlePosition {
  handle: ResizeHandle;
  x: number;
  y: number;
  cursor: string;  // CSS cursor value
}

export interface SelectionState {
  selectedIds: Set<string>;
  boundingBox: { x: number; y: number; width: number; height: number } | null;
}
```

```typescript
// packages/client/src/hooks/useCanvas.ts
export function useCanvas(canvasRef: RefObject<HTMLCanvasElement>) {
  // Handles: mousedown, mousemove, mouseup, wheel (zoom),
  //          touchstart, touchmove, touchend
  // Determines interaction mode: idle, selecting, moving, resizing, rotating, panning
  // Dispatches to TransformManager and EditorStore
}
```

**Testing**:
- `Unit: hitTest on shape -> returns shape when point is inside bounds`
- `Unit: hitTest on overlapping shapes -> returns topmost shape`
- `Unit: hitTest on empty area -> returns null`
- `Unit: hitTest respects rotation -> point inside rotated shape is detected`
- `Unit: moveShapes(["id1"], 10, 20) -> shape.x += 10, shape.y += 20`
- `Unit: resizeShape from SE handle -> width and height increase, x/y unchanged`
- `Unit: resizeShape with preserveAspect -> aspect ratio maintained`
- `Unit: resizeShape from NW handle -> x/y shift, width/height change inversely`
- `Unit: rotateShape by 45 degrees -> rotation updated correctly`
- `Unit: multiselect bounding box -> encompasses all selected shapes`
- `E2E (Playwright): click shape -> selection handles appear; drag shape -> position updates`

---

#### 2.4 — Undo/Redo System

**What**: Implement a command-based undo/redo stack for all canvas operations.

**Design**:

```typescript
// packages/client/src/engine/history.ts
export interface Command {
  type: string;
  execute(): void;
  undo(): void;
  description: string;
}

export class HistoryManager {
  private undoStack: Command[];
  private redoStack: Command[];
  private maxHistory: number;    // default 100

  constructor(maxHistory?: number);

  push(command: Command): void;  // executes command and adds to undo stack
  undo(): void;                  // pops from undo, pushes to redo
  redo(): void;                  // pops from redo, pushes to undo
  canUndo(): boolean;
  canRedo(): boolean;
  clear(): void;
  getHistory(): { description: string; index: number }[];

  // Group multiple commands into a single undoable unit
  startGroup(description: string): void;
  endGroup(): void;
}

// Example commands:
export class MoveCommand implements Command {
  type = 'move';
  constructor(
    private sceneGraph: SceneGraph,
    private shapeIds: string[],
    private dx: number,
    private dy: number,
  ) {}

  execute(): void { /* apply dx, dy */ }
  undo(): void { /* apply -dx, -dy */ }
  get description(): string { return `Move ${this.shapeIds.length} shape(s)`; }
}

export class AddShapeCommand implements Command { /* ... */ }
export class DeleteShapeCommand implements Command { /* ... */ }
export class UpdatePropertyCommand implements Command { /* ... */ }
export class ReorderCommand implements Command { /* ... */ }
```

**Testing**:
- `Unit: push MoveCommand -> shape moved, command on undo stack`
- `Unit: undo after move -> shape returns to original position`
- `Unit: redo after undo -> shape moves again`
- `Unit: new command after undo -> redo stack cleared`
- `Unit: command group (startGroup/endGroup) -> multiple commands undo as one`
- `Unit: stack exceeds maxHistory -> oldest commands discarded`
- `Unit: undo on empty stack -> no-op`

---

## Phase 3: File Management & Persistence

### Purpose
Connect the canvas engine to the API for saving and loading design files. After this phase, users can create files, edit designs, save their work, and reopen files with all shapes intact. This closes the loop between client-side editing and server-side persistence.

### Tasks

#### 3.1 — File & Page CRUD API

**What**: Implement API endpoints for creating, reading, updating, and deleting design files and pages, including page content (JSONB shape tree).

**Design**:

```typescript
// API endpoints
// POST   /api/projects/:projectId/files       -> create file (with default Page 1)
// GET    /api/projects/:projectId/files        -> list files in project
// GET    /api/files/:fileId                    -> get file metadata
// PATCH  /api/files/:fileId                    -> update file metadata (name, description)
// DELETE /api/files/:fileId                    -> soft delete file

// GET    /api/files/:fileId/pages              -> list pages (metadata only, no content)
// GET    /api/files/:fileId/pages/:pageId      -> get page with full content JSONB
// POST   /api/files/:fileId/pages              -> create new page
// PUT    /api/files/:fileId/pages/:pageId      -> save page content (full JSONB replacement)
// PATCH  /api/files/:fileId/pages/:pageId      -> partial page update (metadata only)
// DELETE /api/files/:fileId/pages/:pageId      -> delete page

interface SavePageContentBody {
  content: PageContent;   // full shape tree JSONB
}

interface PageResponse {
  id: string;
  file_id: string;
  name: string;
  sort_order: number;
  content: PageContent;
  viewport: { x: number; y: number; zoom: number };
  updated_at: string;
}
```

```typescript
// packages/api/src/services/file.service.ts
export class FileService {
  async createFile(projectId: string, name: string, userId: string): Promise<FileResponse>;
  async getFile(fileId: string): Promise<FileResponse>;
  async updateFile(fileId: string, updates: Partial<FileUpdate>): Promise<FileResponse>;
  async deleteFile(fileId: string): Promise<void>;
  async listFiles(projectId: string): Promise<FileResponse[]>;
}

export class PageService {
  async getPage(pageId: string): Promise<PageResponse>;
  async savePageContent(pageId: string, content: PageContent): Promise<PageResponse>;
  async createPage(fileId: string, name: string): Promise<PageResponse>;
  async deletePage(pageId: string): Promise<void>;
  async reorderPages(fileId: string, pageIds: string[]): Promise<void>;
}
```

**Testing**:
- `Unit: createFile -> file created with default "Page 1" containing empty content JSONB`
- `Unit: savePageContent with valid PageContent -> content saved, updated_at bumped`
- `Unit: savePageContent with invalid JSONB structure -> 400 Bad Request`
- `Unit: getPage -> returns full content JSONB`
- `Integration: create file -> add shapes client-side -> save -> reload page -> same shapes rendered`
- `Unit: deletePage when only 1 page remains -> 400 Bad Request (file must have at least one page)`
- `Unit: reorderPages -> sort_order updated correctly`

---

#### 3.2 — Auto-save & Version History

**What**: Implement client-side auto-save (debounced), and server-side version snapshots for restoring previous states.

**Design**:

```typescript
// packages/client/src/stores/file.store.ts
interface FileStoreState {
  currentFileId: string | null;
  currentPageId: string | null;
  isDirty: boolean;
  lastSavedAt: Date | null;
  isSaving: boolean;
  autoSaveIntervalMs: number;  // default 5000 (5 seconds)
}

// Auto-save logic:
// 1. On any shape change, set isDirty = true
// 2. Debounce: after 2 seconds of no changes, trigger save
// 3. Periodic: if dirty for > 5 seconds, force save
// 4. On window blur/unload: save immediately if dirty
```

```typescript
// API endpoints for versions
// POST   /api/files/:fileId/versions              -> create named version snapshot
// GET    /api/files/:fileId/versions               -> list versions
// GET    /api/files/:fileId/versions/:versionId    -> get version snapshot
// POST   /api/files/:fileId/versions/:versionId/restore -> restore version

interface CreateVersionBody {
  label: string;   // e.g., "Before brand update", "Final v2"
}

interface VersionResponse {
  id: string;
  file_id: string;
  version_number: number;
  label: string;
  created_by: { id: string; display_name: string };
  created_at: string;
}
```

**Testing**:
- `Unit: shape change sets isDirty to true`
- `Unit: auto-save triggers after debounce period when dirty`
- `Unit: rapid changes -> only one save after debounce settles`
- `Unit: createVersion -> snapshot of all pages' content stored`
- `Unit: restoreVersion -> current page content replaced with snapshot content`
- `Unit: listVersions -> returns versions ordered by version_number descending`
- `E2E: edit shape -> wait 5s -> reload page -> changes persisted`

---

#### 3.3 — Asset Upload & Management

**What**: Implement file upload for images and fonts, stored in S3-compatible storage with metadata in PostgreSQL.

**Design**:

```typescript
// API endpoints
// POST   /api/teams/:teamId/assets/upload     -> upload asset (multipart)
// GET    /api/teams/:teamId/assets             -> list assets (paginated, filterable)
// GET    /api/assets/:assetId                  -> get asset metadata
// GET    /api/assets/:assetId/url              -> get presigned download URL
// DELETE /api/assets/:assetId                  -> delete asset

// POST   /api/teams/:teamId/fonts/upload       -> upload font (OpenType/WOFF2)
// GET    /api/teams/:teamId/fonts              -> list team fonts + system fonts

interface UploadResponse {
  id: string;
  filename: string;
  content_type: string;
  size_bytes: number;
  dimensions: { width: number; height: number } | null;
  url: string;                // presigned URL
}
```

```typescript
// packages/api/src/services/storage.service.ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export class StorageService {
  private s3: S3Client;
  private bucket: string;

  async upload(file: MultipartFile, teamId: string, userId: string): Promise<UploadResponse>;
  async getPresignedUrl(assetId: string, expiresInSecs?: number): Promise<string>;
  async deleteObject(assetId: string): Promise<void>;

  // Image processing on upload:
  // 1. Validate MIME type (image/png, image/jpeg, image/svg+xml, image/webp)
  // 2. Extract dimensions (sharp)
  // 3. Generate thumbnail (256x256)
  // 4. Extract EXIF/ICC metadata
  // 5. Store original + thumbnail in S3
}
```

**Testing**:
- `Unit: upload PNG image -> stored in S3, metadata saved to storage_objects`
- `Unit: upload with invalid MIME type -> 400 Bad Request`
- `Unit: upload exceeding size limit (50MB) -> 413 Payload Too Large`
- `Unit: getPresignedUrl -> returns valid URL with expiry`
- `Unit: upload font (WOFF2) -> font metadata extracted, variants populated`
- `Unit: upload OpenType variable font -> variable_axes JSONB populated`
- `Integration: upload image -> use asset_id in ImageShape fill -> image renders on canvas`

---

## Phase 4: Template System & Drag-and-Drop Editor

### Purpose
Build the template library, template picker, and the full drag-and-drop editing experience with a properties panel. After this phase, users can start from a template, customize it with the properties panel, add/remove shapes, and produce a finished design. This is the first "complete user workflow" milestone.

### Tasks

#### 4.1 — Template Library & Picker

**What**: Implement template CRUD, categorization, and a template picker modal that creates a new file from a selected template.

**Design**:

```typescript
// API endpoints
// GET    /api/templates?category=social-media&subcategory=instagram-post -> list templates
// GET    /api/templates/:templateId         -> get template details
// POST   /api/templates/:templateId/use     -> create new file from template
// POST   /api/teams/:teamId/templates       -> create team template from existing file
// PATCH  /api/templates/:templateId         -> update template metadata
// DELETE /api/templates/:templateId         -> delete template

interface UseTemplateBody {
  project_id: string;
  file_name: string;
  dataset_values?: Record<string, string>;  // autofill values
}

// Template categories:
type TemplateCategory =
  | 'social-media'     // instagram-post, instagram-story, facebook-post, twitter-post, linkedin-post
  | 'presentation'     // 16:9, 4:3
  | 'infographic'      // tall-infographic, wide-infographic
  | 'ad'               // google-display, facebook-ad, banner
  | 'print'            // business-card, a4-flyer, a5-flyer, poster
  | 'video'            // youtube-thumbnail, tiktok-cover
  | 'email'            // email-header, newsletter
  | 'web';             // web-banner, hero-section, og-image

// Preset dimensions (100+ presets):
export const TEMPLATE_PRESETS: Record<string, { width: number; height: number; unit: 'px' }> = {
  'instagram-post': { width: 1080, height: 1080, unit: 'px' },
  'instagram-story': { width: 1080, height: 1920, unit: 'px' },
  'facebook-post': { width: 1200, height: 630, unit: 'px' },
  'twitter-post': { width: 1600, height: 900, unit: 'px' },
  'linkedin-post': { width: 1200, height: 627, unit: 'px' },
  'a4-flyer': { width: 2480, height: 3508, unit: 'px' },     // 210x297mm at 300dpi
  'business-card': { width: 1050, height: 600, unit: 'px' },  // 89x51mm at 300dpi
  'youtube-thumbnail': { width: 1280, height: 720, unit: 'px' },
  // ... 90+ more presets
};
```

**Testing**:
- `Unit: list templates with category filter -> returns only matching templates`
- `Unit: use template -> new file created with cloned page content`
- `Unit: use template with dataset_values -> placeholders replaced in text shapes`
- `Unit: create team template from file -> template metadata saved, source_file_id set`
- `Integration: pick template -> file created -> canvas renders template content`

---

#### 4.2 — Properties Panel

**What**: Implement the right-side properties panel that allows editing selected shape attributes (position, size, fills, strokes, text, effects).

**Design**:

```typescript
// packages/client/src/components/panels/PropertiesPanel.tsx
interface PropertiesPanelProps {
  selectedShapeIds: string[];
  sceneGraph: SceneGraph;
  onUpdate: (shapeId: string, updates: Partial<ShapeNode>) => void;
}

// Panel sections (shown conditionally based on shape type):
// - Transform: x, y, width, height, rotation
// - Fill: color picker, gradient editor, opacity, add/remove fills
// - Stroke: color, width, alignment, dash pattern
// - Text (text shapes only): font family, size, weight, style, alignment, color, line height
// - Effects: drop shadow, inner shadow, blur
// - Image (image shapes only): crop, fit mode
// - Frame (frame shapes only): clip content, corner radius
// - Constraints: horizontal/vertical constraint dropdowns
```

```typescript
// Color picker component interface
interface ColorPickerProps {
  value: string;            // hex color
  onChange: (hex: string) => void;
  showAlpha: boolean;
  brandColors?: BrandColor[];  // quick-access brand colors
}
```

**Testing**:
- `Unit: select rect -> transform, fill, stroke, effects sections shown`
- `Unit: select text -> text section shown in addition to standard sections`
- `Unit: change x value in transform -> shape x updated in scene graph`
- `Unit: add fill -> new fill entry appears, shape renders with fill`
- `Unit: change fill color via picker -> shape fill updates in real-time`
- `Unit: multi-select -> shared properties shown, mixed values indicated`
- `Unit: edit locked shape -> property inputs disabled`
- `E2E: select shape, change color in panel -> canvas reflects change immediately`

---

#### 4.3 — Layers Panel & Shape Tools

**What**: Implement the layers panel (left sidebar) showing the shape tree, and the toolbar with shape creation tools.

**Design**:

```typescript
// packages/client/src/components/panels/LayersPanel.tsx
// Displays:
// - Tree view of shapes matching scene graph hierarchy
// - Drag-and-drop reordering within and between groups/frames
// - Eye icon for visibility toggle
// - Lock icon for lock toggle
// - Double-click to rename

// packages/client/src/components/toolbar/Toolbar.tsx
// Tools:
// - Select (V): default pointer tool
// - Frame (F): draw frame/artboard
// - Rectangle (R): draw rectangle
// - Ellipse (O): draw ellipse
// - Line (L): draw line (path)
// - Text (T): click to create text shape
// - Pen (P): draw vector paths
// - Hand (H): pan viewport
// - Zoom (Z): click to zoom in, alt+click to zoom out

interface ToolState {
  activeTool: 'select' | 'frame' | 'rect' | 'ellipse' | 'line' | 'text'
            | 'pen' | 'hand' | 'zoom';
  toolOptions: Record<string, unknown>;
}
```

**Testing**:
- `Unit: layers panel renders shape tree matching scene graph`
- `Unit: drag shape in layers -> reorders in scene graph`
- `Unit: drag shape into frame -> reparents correctly`
- `Unit: toggle visibility in layers -> shape hidden on canvas`
- `Unit: select Rectangle tool, drag on canvas -> rectangle created at drag coordinates`
- `Unit: select Text tool, click on canvas -> text shape created at click position, text editing mode activated`
- `Unit: keyboard shortcut V -> switches to Select tool`
- `E2E: draw rectangle with tool -> shape appears in layers panel and on canvas`

---

## Phase 5: Brand Kit & Design System

### Purpose
Implement brand kit management, enabling teams to define and enforce brand colors, typography, and logos across all designs. After this phase, teams can create brand kits, apply brand styles with one click, and maintain visual consistency.

### Tasks

#### 5.1 — Brand Kit CRUD

**What**: Implement API and UI for creating and managing brand kits with colors, typography styles, and logos.

**Design**:

```typescript
// API endpoints
// POST   /api/teams/:teamId/brand-kits                -> create brand kit
// GET    /api/teams/:teamId/brand-kits                 -> list brand kits
// GET    /api/brand-kits/:kitId                        -> get brand kit details
// PATCH  /api/brand-kits/:kitId                        -> update brand kit
// DELETE /api/brand-kits/:kitId                        -> delete brand kit
// POST   /api/brand-kits/:kitId/set-default            -> set as team default

interface BrandKitBody {
  name: string;
  colors: BrandColor[];
  typography: BrandTypography[];
  logos: BrandLogo[];
  guidelines: BrandGuidelines;
}

interface BrandColor {
  id: string;
  name: string;              // "Primary", "Secondary", "Accent"
  hex: string;
  color_space: 'srgb' | 'display-p3' | 'cmyk';
  cmyk?: [number, number, number, number];  // for print workflows (ISO 15076-1)
}

interface BrandTypography {
  id: string;
  name: string;              // "Heading", "Subheading", "Body", "Caption"
  font_id: string;
  font_family: string;
  font_size: number;
  font_weight: number;
  line_height: number;
  letter_spacing: number;
}

interface BrandLogo {
  id: string;
  name: string;
  variant: 'primary' | 'secondary' | 'icon' | 'monochrome';
  asset_id: string;
}

interface BrandGuidelines {
  min_logo_clearance_px: number;
  forbidden_colors: string[];
  tone_of_voice: string;
}
```

**Testing**:
- `Unit: create brand kit -> kit created with colors, typography, logos JSONB`
- `Unit: update brand kit colors -> colors JSONB replaced`
- `Unit: set default brand kit -> previous default unset, new default set`
- `Unit: delete brand kit that is default -> 400 Bad Request`
- `Unit: list brand kits -> returns all kits for team`
- `Unit: brand kit with CMYK color -> cmyk array stored correctly`

---

#### 5.2 — Brand Application in Editor

**What**: Integrate brand kits into the editor: brand colors in color picker, brand typography in text panel, one-click brand application to designs.

**Design**:

```typescript
// packages/client/src/stores/brand.store.ts
interface BrandStoreState {
  activeBrandKit: BrandKit | null;
  teamBrandKits: BrandKit[];

  // Actions
  loadBrandKits(teamId: string): Promise<void>;
  setActiveBrandKit(kitId: string): void;

  // Apply brand to selection
  applyBrandColor(shapeIds: string[], colorId: string,
                  target: 'fill' | 'stroke' | 'text'): void;
  applyBrandTypography(shapeIds: string[], typographyId: string): void;

  // One-click full brand application
  applyBrandToFile(fileId: string, brandKitId: string): Promise<void>;
}

// Color picker integration:
// - "Brand Colors" section at top of color picker
// - Clicking brand color applies it directly
// - Non-brand colors flagged with subtle indicator

// Text panel integration:
// - "Brand Styles" dropdown at top of text section
// - Selecting brand typography applies font, size, weight, line height simultaneously
```

**Testing**:
- `Unit: brand colors appear in color picker when brand kit is active`
- `Unit: applyBrandColor -> shape fill updated to brand color hex`
- `Unit: applyBrandTypography -> text shape font/size/weight all updated`
- `Unit: applyBrandToFile -> all text shapes use brand typography, solid fills use nearest brand color`
- `Unit: no active brand kit -> brand sections hidden in panels`
- `E2E: create brand kit -> open editor -> brand colors visible in color picker`

---

## Phase 6: Real-Time Collaboration

### Purpose
Enable multiple users to edit the same design file simultaneously with live cursor tracking, selection awareness, and conflict-free concurrent editing. This is the feature that makes the tool viable for teams.

### Tasks

#### 6.1 — WebSocket Infrastructure

**What**: Implement WebSocket server for real-time collaboration, including connection management, room-based file sessions, and event broadcasting.

**Design**:

```typescript
// packages/api/src/ws/collaboration.ts
import { WebSocket } from 'ws';

interface CollaborationSession {
  fileId: string;
  userId: string;
  displayName: string;
  color: string;          // assigned collaborator color
  cursorPosition: { x: number; y: number; pageId: string } | null;
  selection: string[];    // selected shape IDs
  connectedAt: Date;
}

// WebSocket message types (client -> server):
type ClientMessage =
  | { type: 'cursor_move'; data: { x: number; y: number; pageId: string } }
  | { type: 'selection_change'; data: { shapeIds: string[] } }
  | { type: 'shape_update'; data: { operations: ShapeOperation[] } }
  | { type: 'page_switch'; data: { pageId: string } };

// WebSocket message types (server -> client):
type ServerMessage =
  | { type: 'user_joined'; data: CollaborationSession }
  | { type: 'user_left'; data: { userId: string } }
  | { type: 'cursor_moved'; data: { userId: string; x: number; y: number; pageId: string } }
  | { type: 'selection_changed'; data: { userId: string; shapeIds: string[] } }
  | { type: 'shapes_updated'; data: { userId: string; operations: ShapeOperation[] } }
  | { type: 'active_users'; data: CollaborationSession[] };

type ShapeOperation =
  | { op: 'add'; shape: ShapeNode; parentId: string | null; index: number }
  | { op: 'update'; shapeId: string; changes: Partial<ShapeNode> }
  | { op: 'delete'; shapeId: string }
  | { op: 'reorder'; shapeId: string; newIndex: number }
  | { op: 'reparent'; shapeId: string; newParentId: string; index: number };
```

Cursor positions and selection state stored in Redis for fast access. Shape operations broadcast to all other users in the file room and persisted via the page save mechanism.

**Testing**:
- `Unit: WebSocket connect with valid auth token -> joins file room`
- `Unit: WebSocket connect with invalid token -> connection rejected`
- `Unit: cursor_move message -> broadcast to other users in room`
- `Unit: shape_update message -> operations applied to scene graph, broadcast to others`
- `Unit: user disconnect -> user_left broadcast, session cleaned from Redis`
- `Integration: two clients connected -> client A moves shape -> client B receives update`
- `Integration: client reconnects -> receives current file state and active users`

---

#### 6.2 — Presence & Collaboration UI

**What**: Implement the client-side collaboration UI: avatar bar, live cursors, selection indicators, and conflict resolution.

**Design**:

```typescript
// packages/client/src/stores/collaboration.store.ts
interface CollaborationStoreState {
  isConnected: boolean;
  activeUsers: Map<string, CollaboratorState>;
  localUserId: string;

  connect(fileId: string): void;
  disconnect(): void;
  sendCursorMove(x: number, y: number, pageId: string): void;
  sendSelectionChange(shapeIds: string[]): void;
  sendShapeOperations(operations: ShapeOperation[]): void;
}

interface CollaboratorState {
  userId: string;
  displayName: string;
  avatarUrl: string;
  color: string;             // "#3B82F6" etc., assigned per session
  cursor: { x: number; y: number; pageId: string } | null;
  selection: string[];
}

// Conflict resolution strategy: Last-Writer-Wins with operational transform
// 1. Each operation includes a monotonic sequence number
// 2. Server orders operations by arrival time
// 3. Conflicting updates to the same shape property -> last write wins
// 4. Structural operations (add/delete/reparent) -> server is source of truth
```

**Testing**:
- `Unit: collaborator cursor renders at correct canvas position`
- `Unit: collaborator cursor label shows display name`
- `Unit: collaborator selects shape -> shape outline drawn in collaborator's color`
- `Unit: avatar bar shows all active users`
- `Unit: user leaves -> avatar removed, cursor disappears`
- `E2E (two browser contexts): user A drags shape -> user B sees shape move in real-time`

---

#### 6.3 — Comments & Annotations

**What**: Implement in-canvas commenting with threaded replies and resolution.

**Design**:

```typescript
// API endpoints
// POST   /api/files/:fileId/comments          -> create comment (pinned to canvas position)
// GET    /api/files/:fileId/comments           -> list comments
// POST   /api/comments/:commentId/replies      -> reply to comment
// PATCH  /api/comments/:commentId/resolve       -> resolve comment
// DELETE /api/comments/:commentId              -> delete comment

interface CreateCommentBody {
  page_id: string;
  content: string;
  position: { x: number; y: number; shape_id?: string };
}

interface CommentResponse {
  id: string;
  content: string;
  user: { id: string; display_name: string; avatar_url: string };
  position: { x: number; y: number; shape_id?: string };
  resolved: boolean;
  replies: CommentResponse[];
  created_at: string;
}
```

**Testing**:
- `Unit: create comment at canvas position -> comment pin rendered`
- `Unit: click comment pin -> comment thread opens`
- `Unit: reply to comment -> reply appears in thread`
- `Unit: resolve comment -> pin changes to resolved state`
- `Unit: comment on shape -> pin moves with shape`
- `Integration: user A creates comment -> user B sees comment appear (via WebSocket)`

---

## Phase 7: AI Generation Engine

### Purpose
Integrate AI capabilities for text-to-layout generation, text-to-image generation, and AI-powered copy generation. This phase delivers the core AI-native differentiator described in the research and README. After this phase, users can generate complete designs, individual images, and on-brand copy through natural language prompts.

### Tasks

#### 7.1 — AI Provider Adapter Layer

**What**: Build a provider-agnostic adapter layer that abstracts LLM and image generation APIs.

**Design**:

```typescript
// packages/api/src/services/ai/provider.ts
export interface AIProvider {
  name: string;
  generateStructuredOutput<T>(
    systemPrompt: string,
    userPrompt: string,
    schema: ZodSchema<T>,
    options?: GenerationOptions,
  ): Promise<T>;

  generateImage(
    prompt: string,
    options: ImageGenerationOptions,
  ): Promise<GeneratedImage>;

  generateText(
    systemPrompt: string,
    userPrompt: string,
    options?: TextGenerationOptions,
  ): Promise<string>;
}

export interface GenerationOptions {
  model?: string;
  temperature?: number;
  maxTokens?: number;
}

export interface ImageGenerationOptions {
  width: number;
  height: number;
  style?: 'natural' | 'vivid';
  count?: number;            // number of variations
}

export interface GeneratedImage {
  url: string;
  revisedPrompt: string;
}

// Provider implementations:
export class OpenAIProvider implements AIProvider { /* GPT-4o + DALL-E 3 */ }
export class AnthropicProvider implements AIProvider { /* Claude for text/structured output */ }

// Factory:
export function createProvider(config: AIConfig): AIProvider;
```

```typescript
// packages/api/src/services/ai/config.ts
export interface AIConfig {
  textProvider: 'openai' | 'anthropic';
  imageProvider: 'openai';           // DALL-E 3
  openaiApiKey: string;
  anthropicApiKey?: string;
  defaultTextModel: string;          // 'gpt-4o' or 'claude-sonnet-4-20250514'
  defaultImageModel: string;         // 'dall-e-3'
  rateLimits: {
    requestsPerMinute: number;
    tokensPerDay: number;
  };
}
```

**Testing**:
- `Unit: OpenAIProvider.generateStructuredOutput -> returns parsed Zod schema`
- `Unit: OpenAIProvider.generateImage -> returns URL and revised prompt`
- `Unit: provider with invalid API key -> throws AuthenticationError`
- `Unit: provider with rate limit exceeded -> throws RateLimitError with retry-after`
- `Integration (mocked API): generateStructuredOutput with layout schema -> valid ShapeNode[] returned`

---

#### 7.2 — Text-to-Layout Generation

**What**: Implement AI-powered layout generation that converts a text prompt into a complete page design with positioned shapes, styled text, and color schemes.

**Design**:

```typescript
// packages/api/src/services/ai-generation.service.ts
export class AIGenerationService {
  async generateLayout(
    prompt: string,
    options: LayoutGenerationOptions,
  ): Promise<GeneratedLayout>;
}

interface LayoutGenerationOptions {
  width: number;
  height: number;
  brandKitId?: string;       // apply brand colors/fonts
  category?: string;         // 'social-media', 'presentation', 'ad'
  style?: string;            // 'minimal', 'bold', 'corporate', 'playful'
}

interface GeneratedLayout {
  shapes: ShapeNode[];
  reasoning: string;         // AI's design rationale
}

// System prompt for layout generation (excerpt):
// "You are a professional graphic designer. Given a design brief, generate a layout
//  as a JSON array of shape objects. Each shape must include:
//  - type, name, x, y, width, height
//  - fills (with colors), strokes, text content where applicable
//  - parent_id for nesting shapes within frames
//
//  Design principles to follow:
//  - Visual hierarchy: headlines largest, body smaller, CTAs prominent
//  - White space: leave breathing room (minimum 10% margins)
//  - Alignment: use consistent alignment grids
//  - Color: limit to 3-4 colors maximum, ensure WCAG AA contrast (4.5:1 for text)
//  - Typography: maximum 2 font families
//
//  Output format: JSON array matching the ShapeNode schema."

// The structured output schema ensures LLM returns valid ShapeNode[]:
const LayoutOutputSchema = z.array(ShapeNodeSchema);
```

```typescript
// packages/api/src/routes/ai.ts
// POST /api/ai/generate-layout
interface GenerateLayoutBody {
  prompt: string;
  file_id: string;
  page_id: string;
  options: LayoutGenerationOptions;
}

// Response: async job
interface GenerateLayoutResponse {
  job_id: string;
  status: 'queued';
}

// BullMQ worker processes the job:
// 1. Load brand kit if specified
// 2. Build system prompt with brand constraints
// 3. Call AI provider for structured output
// 4. Validate generated shapes against ShapeNode schema
// 5. Add shapes to page content
// 6. Update job status to 'completed'
```

**Testing**:
- `Unit: generateLayout with prompt "Instagram post for coffee shop" -> returns valid ShapeNode array`
- `Unit: generated layout respects width/height constraints`
- `Unit: generated layout with brandKit -> uses brand colors and fonts`
- `Unit: invalid LLM output (malformed JSON) -> job marked as failed, user notified`
- `Integration (mocked LLM): generate layout -> shapes added to page -> canvas renders generated design`
- `Unit: WCAG contrast check on generated text/background combinations -> all pass AA (4.5:1)`
- `Fixture: test with 5 common prompts ("business card", "instagram story", "presentation slide", "facebook ad", "email header") -> all produce valid layouts`

---

#### 7.3 — Text-to-Image Generation

**What**: Implement AI image generation (DALL-E 3) integrated into the editor, allowing users to generate images and place them on the canvas.

**Design**:

```typescript
// API endpoint
// POST /api/ai/generate-image
interface GenerateImageBody {
  prompt: string;
  file_id: string;
  width: number;          // target width
  height: number;         // target height
  style: 'natural' | 'vivid';
  count: number;          // 1-4 variations
}

interface GenerateImageResponse {
  job_id: string;
  status: 'queued';
}

// Worker flow:
// 1. Call DALL-E 3 API with prompt
// 2. Download generated image
// 3. Upload to S3 storage
// 4. Create storage_object record
// 5. Return asset_id for placement on canvas
```

**Testing**:
- `Unit: generate image with valid prompt -> job created, status 'queued'`
- `Integration (mocked DALL-E): generate image -> image saved to S3 -> asset_id returned`
- `Unit: generate image with NSFW prompt -> content policy error returned`
- `Unit: image uploaded to S3 -> dimensions extracted, thumbnail generated`
- `E2E: generate image -> place on canvas -> image renders correctly`

---

#### 7.4 — AI Copy Generation

**What**: Implement AI-powered text generation for headlines, body copy, and CTAs, with tone and length adaptation.

**Design**:

```typescript
// API endpoint
// POST /api/ai/generate-copy
interface GenerateCopyBody {
  prompt: string;               // "Write a headline for a coffee shop promotion"
  context?: string;             // existing design context
  tone?: 'professional' | 'casual' | 'bold' | 'playful' | 'formal';
  max_length?: number;          // character limit
  language?: string;            // ISO 639-1 code
  variations?: number;          // number of alternatives (1-5)
}

interface GenerateCopyResponse {
  copies: {
    text: string;
    tone: string;
    character_count: number;
  }[];
}

// Also: inline adaptation
// POST /api/ai/adapt-copy
interface AdaptCopyBody {
  text: string;
  target_length: number;        // shrink or expand to fit layout
  target_tone?: string;
  target_language?: string;
}
```

**Testing**:
- `Unit: generate copy -> returns text matching requested tone`
- `Unit: generate copy with max_length 50 -> all variations under 50 characters`
- `Unit: generate copy with language 'es' -> Spanish text returned`
- `Unit: adapt copy to shorter length -> meaning preserved, text shortened`
- `Unit: generate 3 variations -> 3 distinct alternatives returned`

---

## Phase 8: Export System

### Purpose
Implement multi-format export supporting PNG, JPEG, SVG, PDF, WebP, and AVIF, including print-ready workflows with CMYK color profiles and PDF/X output (ISO 32000-2). After this phase, users can export their designs for web, social media, and professional print production.

### Tasks

#### 8.1 — Raster Export (PNG, JPEG, WebP, AVIF)

**What**: Implement server-side rendering of design pages to raster image formats with quality and scale controls.

**Design**:

```typescript
// API endpoint
// POST /api/files/:fileId/export
interface ExportBody {
  page_ids: string[];           // which pages to export
  format: 'png' | 'jpeg' | 'webp' | 'avif' | 'svg' | 'pdf';
  scale: number;                // 1x, 2x, 3x
  quality?: number;             // 1-100 for lossy formats
  background?: string;          // hex color, or 'transparent' for PNG/WebP
}

interface ExportResponse {
  job_id: string;
  status: 'queued';
}

// Worker flow (raster):
// 1. Load page content from database
// 2. Render shapes to a headless canvas (using node-canvas or sharp)
// 3. Export at target resolution (width * scale, height * scale)
// 4. Encode to target format with quality setting
// 5. Upload to S3 with expiring download URL
// 6. Update job status to 'completed' with download URL
```

```typescript
// packages/worker/src/workers/export.worker.ts
import sharp from 'sharp';

export class RasterExporter {
  async exportPage(
    content: PageContent,
    format: 'png' | 'jpeg' | 'webp' | 'avif',
    options: {
      width: number;
      height: number;
      scale: number;
      quality: number;
      background: string;
    },
  ): Promise<Buffer>;

  // Uses sharp for final encoding:
  // PNG: lossless, supports transparency
  // JPEG: lossy, quality 1-100, no transparency
  // WebP: lossy or lossless, supports transparency
  // AVIF: lossy, best compression ratio
}
```

**Testing**:
- `Unit: export page as PNG -> valid PNG buffer with correct dimensions`
- `Unit: export at 2x scale -> dimensions doubled`
- `Unit: export JPEG with quality 80 -> file size smaller than quality 100`
- `Unit: export PNG with transparent background -> alpha channel preserved`
- `Unit: export WebP -> valid WebP with smaller file size than PNG`
- `Fixture: export test page with rect, text, image shapes -> visual regression test passes`

---

#### 8.2 — SVG Export

**What**: Implement export to W3C SVG 2 format with correct mapping from shapes to SVG elements.

**Design**:

```typescript
// packages/worker/src/workers/svg-exporter.ts
export class SVGExporter {
  exportPage(content: PageContent, options: SVGExportOptions): string;

  // Shape -> SVG mapping:
  // frame/rect -> <rect> with rx/ry for corner radius
  // ellipse   -> <ellipse> cx/cy/rx/ry
  // text      -> <text> with <tspan> for styled runs
  // path      -> <path d="...">
  // image     -> <image href="..." x y width height>
  // group     -> <g transform="...">
  // fills     -> <defs><linearGradient>/<radialGradient> referenced by fill="url(#...)"
  // effects   -> <filter> elements for drop-shadow, blur
  // opacity   -> opacity attribute
  // rotation  -> transform="rotate(deg, cx, cy)"
}

interface SVGExportOptions {
  embedImages: boolean;      // base64 embed vs. external references
  embedFonts: boolean;       // @font-face with base64 WOFF2
  minify: boolean;
}
```

**Testing**:
- `Unit: rect shape -> <rect> element with correct x, y, width, height, fill`
- `Unit: rect with corner radius -> rx/ry attributes set`
- `Unit: text shape -> <text> with correct font-family, font-size, fill`
- `Unit: gradient fill -> <linearGradient> in <defs>, referenced by fill="url(#...)"`
- `Unit: group with rotation -> <g transform="rotate(...)"> wrapping children`
- `Unit: embedImages=true -> image data as base64 data URI`
- `Unit: valid SVG output passes W3C SVG validator`

---

#### 8.3 — PDF Export with Print-Ready Workflows

**What**: Implement PDF export compliant with ISO 32000-2, including PDF/X for print production, CMYK color spaces, ICC profile embedding, and bleed/crop marks.

**Design**:

```typescript
// packages/worker/src/workers/pdf-exporter.ts
import PDFDocument from 'pdfkit';

export class PDFExporter {
  async exportPage(
    content: PageContent,
    options: PDFExportOptions,
  ): Promise<Buffer>;
}

interface PDFExportOptions {
  colorSpace: 'srgb' | 'cmyk';          // ISO 15076-1
  pdfStandard?: 'pdf-x-1a' | 'pdf-x-4'; // ISO 32000-2
  iccProfilePath?: string;               // path to ICC v4 profile
  includeBleed: boolean;
  bleedMm: number;                       // typically 3mm
  includeCropMarks: boolean;
  includeRegistrationMarks: boolean;
  embedFonts: boolean;                   // always true for PDF/X
  quality: number;                       // image resampling quality
}

// CMYK conversion:
// sRGB hex -> Lab (D50) -> CMYK via ICC profile
// Uses littlecms (via native addon) or color-convert for basic conversion
```

**Testing**:
- `Unit: PDF export in sRGB -> valid PDF with RGB color space`
- `Unit: PDF export in CMYK -> colors converted, CMYK color space declared`
- `Unit: PDF/X-4 export -> output intent, ICC profile embedded, fonts embedded`
- `Unit: bleed included -> page size expanded by bleed amount, crop marks drawn`
- `Unit: text with embedded fonts -> fonts subset and embedded in PDF`
- `Unit: images resampled at target DPI (300 for print)`
- `Fixture: export test design as PDF/X-4 -> validate with PDF/A validator tool`

---

## Phase 9: Accessibility Engine

### Purpose
Embed WCAG 2.2 accessibility checking directly into the design workflow. After this phase, designers get real-time feedback on contrast ratios, missing alt text, and reading order issues before they publish. This addresses a gap that no competitor has filled (per the research findings) and is essential for EU EAA compliance (enforced June 2025).

### Tasks

#### 9.1 — WCAG Contrast Checker

**What**: Implement real-time color contrast checking against WCAG 2.2 success criteria.

**Design**:

```typescript
// packages/shared/src/utils/accessibility.ts

export interface ContrastResult {
  ratio: number;              // e.g., 4.87
  passesAA: boolean;          // >= 4.5:1 for normal text, >= 3:1 for large text
  passesAAA: boolean;         // >= 7:1 for normal text, >= 4.5:1 for large text
  foreground: string;         // hex
  background: string;         // hex
  isLargeText: boolean;       // >= 18pt or >= 14pt bold (WCAG definition)
}

export function calculateContrastRatio(fg: string, bg: string): number;
export function getRelativeLuminance(hex: string): number;

export function checkTextContrast(
  textShape: TextShape,
  backgroundShapes: ShapeNode[],
  pageBackground: string,
): ContrastResult;

// WCAG 2.2 thresholds:
// AA normal text: 4.5:1
// AA large text (>=18pt or >=14pt bold): 3:1
// AAA normal text: 7:1
// AAA large text: 4.5:1
```

```typescript
// packages/api/src/services/accessibility.service.ts
export class AccessibilityService {
  checkPage(pageContent: PageContent, wcagLevel: 'A' | 'AA' | 'AAA'): AccessibilityReport;
}

export interface AccessibilityReport {
  level: 'A' | 'AA' | 'AAA';
  passed: boolean;
  issues: AccessibilityIssue[];
  summary: {
    total_issues: number;
    contrast_issues: number;
    alt_text_issues: number;
    reading_order_issues: number;
  };
}

export interface AccessibilityIssue {
  type: 'contrast' | 'alt-text' | 'reading-order' | 'color-only';
  severity: 'error' | 'warning';
  shape_id: string;
  shape_name: string;
  message: string;
  wcag_criterion: string;    // e.g., "1.4.3 Contrast (Minimum)"
  details: Record<string, unknown>;
  suggestion?: string;
}
```

**Testing**:
- `Unit: black text on white background -> ratio 21:1, passes AA and AAA`
- `Unit: light gray text on white background -> ratio below 4.5:1, fails AA`
- `Unit: 18pt text with 3.5:1 ratio -> passes AA (large text threshold)`
- `Unit: 14pt bold text with 3.5:1 ratio -> passes AA (large text)`
- `Unit: 14pt normal text with 3.5:1 ratio -> fails AA (not large text)`
- `Unit: text over gradient background -> uses average luminance of gradient stops`
- `Unit: checkPage with mix of passing/failing text -> report lists all issues with WCAG criterion`

---

#### 9.2 — Alt Text & Reading Order

**What**: Implement alt text suggestions for images and reading order validation for screen reader compatibility.

**Design**:

```typescript
// Alt text: stored in shape properties
// ImageShape.properties.alt_text: string

// API endpoint for AI-generated alt text:
// POST /api/ai/generate-alt-text
interface GenerateAltTextBody {
  asset_id: string;          // image to describe
  context?: string;          // surrounding design context
}

interface GenerateAltTextResponse {
  alt_text: string;
  confidence: number;
}

// Reading order: determined by shape position (top-to-bottom, left-to-right)
// Stored as optional metadata: page.content.reading_order: string[]

export function inferReadingOrder(shapes: ShapeNode[]): string[];
// Algorithm:
// 1. Filter to text and image shapes only
// 2. Sort by y position (top to bottom), then x position (left to right)
// 3. Group shapes that overlap vertically (same "line")
// 4. Within each group, sort left to right
// 5. Return ordered array of shape IDs
```

**Testing**:
- `Unit: image without alt text -> accessibility issue flagged`
- `Unit: decorative image with alt_text="" -> no issue (decorative image is valid)`
- `Unit: generate alt text for photo -> descriptive text returned`
- `Unit: inferReadingOrder for simple top-to-bottom layout -> correct order`
- `Unit: inferReadingOrder for two-column layout -> columns read top-to-bottom independently`
- `Unit: reading order validation with crossed reading paths -> warning issued`

---

## Phase 10: Multi-Platform Resize & Social Publishing

### Purpose
Enable intelligent multi-platform resizing and direct publishing to social media platforms. After this phase, a single design can be exported as optimized variants for Instagram, Facebook, Twitter, LinkedIn, YouTube, and email in one operation.

### Tasks

#### 10.1 — Intelligent Resize Engine

**What**: Implement a resize engine that adapts layouts to different dimensions while maintaining visual hierarchy and readability.

**Design**:

```typescript
// packages/api/src/services/resize.service.ts
export class ResizeService {
  resizeLayout(
    content: PageContent,
    sourceDimensions: { width: number; height: number },
    targetDimensions: { width: number; height: number },
    options: ResizeOptions,
  ): PageContent;
}

interface ResizeOptions {
  strategy: 'scale' | 'reflow' | 'ai-optimize';
  // 'scale': proportional scaling (simple, fast)
  // 'reflow': repositions elements using constraint system
  // 'ai-optimize': AI re-layouts for optimal composition at target size
  preserveTextSize: boolean;  // keep text readable rather than scaling
  brandKitId?: string;        // maintain brand compliance during resize
}

// Constraint-based reflow algorithm:
// 1. Identify anchor shapes (largest text, primary image, CTA button)
// 2. Calculate relative positions as percentages of source dimensions
// 3. Map to target dimensions using constraint rules
// 4. Adjust font sizes to maintain readability (minimum 12px)
// 5. Reflow text shapes if content overflows new bounds
// 6. Scale decorative elements proportionally
```

```typescript
// API endpoint
// POST /api/files/:fileId/resize
interface ResizeBody {
  page_id: string;
  targets: { preset: string; name: string }[];
  // e.g., [
  //   { preset: 'instagram-post', name: 'IG Post' },
  //   { preset: 'facebook-post', name: 'FB Post' },
  //   { preset: 'twitter-post', name: 'Twitter' }
  // ]
  strategy: 'scale' | 'reflow' | 'ai-optimize';
}

interface ResizeResponse {
  generated_pages: { page_id: string; preset: string; name: string }[];
}
```

**Testing**:
- `Unit: scale resize 1080x1080 -> 1200x630 -> all shapes proportionally scaled`
- `Unit: reflow resize -> text shapes maintain minimum 12px font size`
- `Unit: reflow resize -> shapes with constraint 'center' remain centered`
- `Unit: resize to multiple targets -> creates one new page per target`
- `Unit: preserveTextSize=true -> text font sizes unchanged, positions adjusted`
- `Fixture: resize test design to 5 presets -> visual review of all outputs`

---

#### 10.2 — Social Platform Publishing

**What**: Implement direct publishing to social media platforms via their APIs.

**Design**:

```typescript
// API endpoints
// POST /api/integrations/connect/:platform     -> OAuth connect to platform
// POST /api/files/:fileId/publish              -> publish to connected platforms
// GET  /api/integrations/status                -> list connected platforms

type SocialPlatform = 'instagram' | 'facebook' | 'twitter' | 'linkedin' | 'pinterest';

interface PublishBody {
  page_id: string;
  platforms: {
    platform: SocialPlatform;
    caption: string;
    scheduled_at?: string;    // ISO 8601 for scheduled posts
    hashtags?: string[];
  }[];
}

interface PublishResponse {
  results: {
    platform: string;
    status: 'published' | 'scheduled' | 'failed';
    post_url?: string;
    error?: string;
  }[];
}
```

**Testing**:
- `Integration (mocked API): publish to Instagram -> image uploaded, caption set`
- `Integration (mocked API): publish to Facebook -> post created with image`
- `Unit: scheduled publish -> job created in BullMQ with delay`
- `Unit: publish without connected platform -> 400 Bad Request with setup instructions`
- `Unit: image automatically exported at correct dimensions for each platform`

---

## Phase 11: Design-to-Code Export

### Purpose
Enable developers to extract design tokens, CSS, and React component code from designs. Implements W3C DTCG design token format for standards-compliant token pipelines. After this phase, the tool bridges the designer-developer handoff gap.

### Tasks

#### 11.1 — Design Token Export (DTCG Format)

**What**: Export brand kits and design properties as W3C Design Tokens Community Group (DTCG) JSON format, compatible with Style Dictionary and Theo pipelines.

**Design**:

```typescript
// packages/api/src/services/token-export.service.ts
export class TokenExportService {
  exportBrandKitAsTokens(brandKitId: string): DTCGTokenFile;
  exportFileAsTokens(fileId: string): DTCGTokenFile;
}

// W3C DTCG format:
interface DTCGTokenFile {
  [groupName: string]: DTCGTokenGroup;
}

interface DTCGTokenGroup {
  $type?: string;
  $description?: string;
  [tokenName: string]: DTCGToken | DTCGTokenGroup;
}

interface DTCGToken {
  $value: string | number | Record<string, unknown>;
  $type: 'color' | 'dimension' | 'fontFamily' | 'fontWeight'
       | 'duration' | 'cubicBezier' | 'shadow';
  $description?: string;
}

// Example output:
// {
//   "color": {
//     "primary": { "$value": "#FF5733", "$type": "color",
//                  "$description": "Primary brand color" },
//     "secondary": { "$value": "#1A1A2E", "$type": "color" }
//   },
//   "typography": {
//     "heading": {
//       "fontFamily": { "$value": "Inter", "$type": "fontFamily" },
//       "fontSize": { "$value": "24px", "$type": "dimension" },
//       "fontWeight": { "$value": 700, "$type": "fontWeight" },
//       "lineHeight": { "$value": "1.2", "$type": "dimension" }
//     }
//   }
// }
```

**Testing**:
- `Unit: brand kit with 3 colors -> DTCG JSON with 3 color tokens`
- `Unit: brand kit with typography -> fontFamily, fontSize, fontWeight tokens`
- `Unit: output validates against DTCG JSON schema`
- `Unit: exported tokens importable by Style Dictionary -> CSS custom properties generated`

---

#### 11.2 — CSS & React Export

**What**: Generate CSS and React component code from selected shapes or components.

**Design**:

```typescript
// API endpoint
// POST /api/files/:fileId/export-code
interface ExportCodeBody {
  page_id: string;
  shape_ids: string[];
  format: 'css' | 'react' | 'tailwind';
}

interface ExportCodeResponse {
  files: {
    filename: string;
    content: string;
    language: 'css' | 'tsx' | 'json';
  }[];
}

// CSS generation maps shape properties to CSS:
// x, y -> position: absolute; left/top
// width, height -> width/height
// fills[0].color -> background-color
// corner_radius -> border-radius
// text.font_family -> font-family
// text.font_size -> font-size
// strokes -> border
// effects.drop-shadow -> box-shadow
// opacity -> opacity
```

**Testing**:
- `Unit: rect with fill -> CSS with background-color, width, height, border-radius`
- `Unit: text shape -> CSS with font-family, font-size, color, text-align`
- `Unit: React export -> functional component with inline styles`
- `Unit: Tailwind export -> className string with Tailwind utilities`
- `Unit: nested frame -> parent-child div structure in React/HTML`

---

## Phase 12: Enterprise Features & Hardening

### Purpose
Add enterprise-grade features: team approval workflows, audit logging, SAML SSO, and performance optimization. After this phase, the platform is production-ready for enterprise deployments with compliance and governance requirements.

### Tasks

#### 12.1 — Approval Workflows

**What**: Implement formal review and approval gates for designs before publication.

**Design**:

```typescript
// API endpoints
// POST /api/files/:fileId/reviews           -> submit for review
// GET  /api/files/:fileId/reviews           -> list reviews
// POST /api/reviews/:reviewId/approve        -> approve
// POST /api/reviews/:reviewId/reject         -> reject with feedback
// POST /api/reviews/:reviewId/lock           -> lock version after approval

interface SubmitReviewBody {
  reviewer_ids: string[];
  message: string;
  version_id: string;        // specific version under review
}

type ReviewStatus = 'pending' | 'approved' | 'rejected' | 'locked';

interface ReviewResponse {
  id: string;
  file_id: string;
  version_id: string;
  status: ReviewStatus;
  submitted_by: UserSummary;
  reviewers: {
    user: UserSummary;
    decision: 'pending' | 'approved' | 'rejected';
    comment?: string;
  }[];
  created_at: string;
}
```

**Testing**:
- `Unit: submit review -> review created with pending status for each reviewer`
- `Unit: all reviewers approve -> status changes to 'approved'`
- `Unit: any reviewer rejects -> status changes to 'rejected'`
- `Unit: lock approved version -> file version becomes immutable`
- `Unit: edit locked version -> new version created, lock preserved on original`
- `Unit: non-reviewer attempts to approve -> 403 Forbidden`

---

#### 12.2 — SAML SSO & Enterprise Auth

**What**: Implement SAML 2.0 Single Sign-On for enterprise identity provider integration.

**Design**:

```typescript
// API endpoints
// POST /api/teams/:teamId/sso/configure       -> configure SAML provider
// GET  /api/auth/saml/:teamSlug/login          -> initiate SAML flow
// POST /api/auth/saml/:teamSlug/callback       -> SAML assertion consumer service

interface SAMLConfiguration {
  team_id: string;
  idp_entity_id: string;
  idp_sso_url: string;
  idp_certificate: string;     // X.509 certificate for signature validation
  sp_entity_id: string;        // auto-generated
  attribute_mapping: {
    email: string;             // SAML attribute name for email
    display_name: string;
    groups?: string;           // for automatic role assignment
  };
}
```

**Testing**:
- `Unit: configure SAML -> SSO config saved for team`
- `Integration (mocked IdP): SAML login -> redirect to IdP, assertion parsed, user created/linked`
- `Unit: invalid SAML signature -> 401 Unauthorized`
- `Unit: SAML assertion with new user -> user auto-provisioned in team`
- `Unit: SAML assertion with existing user -> session created without new user`

---

#### 12.3 — Audit Logging

**What**: Implement comprehensive audit logging for compliance-sensitive operations.

**Design**:

```typescript
// Audit log table (append-only)
// CREATE TABLE audit_logs (
//     id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
//     team_id         UUID NOT NULL,
//     actor_id        UUID NOT NULL,
//     action          VARCHAR(100) NOT NULL,
//     resource_type   VARCHAR(50) NOT NULL,
//     resource_id     UUID,
//     details         JSONB DEFAULT '{}',
//     ip_address      INET,
//     user_agent      TEXT,
//     created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
// );

type AuditAction =
  | 'user.login' | 'user.logout'
  | 'team.member_added' | 'team.member_removed' | 'team.role_changed'
  | 'file.created' | 'file.deleted' | 'file.exported'
  | 'brand_kit.updated'
  | 'review.submitted' | 'review.approved' | 'review.rejected'
  | 'sso.configured'
  | 'api_key.created' | 'api_key.revoked';

// API endpoint
// GET /api/teams/:teamId/audit-log?action=file.exported&from=2026-01-01&to=2026-05-25
```

**Testing**:
- `Unit: user login -> audit log entry created with actor, IP, user agent`
- `Unit: file export -> audit log entry with file_id and format details`
- `Unit: audit log is append-only -> no UPDATE/DELETE operations permitted`
- `Unit: query audit log with filters -> returns matching entries paginated`
- `Integration: complete workflow (login -> create file -> export) -> 3 audit entries`

---

#### 12.4 — Performance Optimization

**What**: Optimize canvas rendering, API response times, and database queries for production-scale usage.

**Design**:

```
Performance targets:
- Canvas: 60fps rendering with 1,000+ shapes on page
- API: < 200ms p95 latency for page load (including JSONB content)
- WebSocket: < 50ms latency for cursor updates between collaborators
- Export: < 10s for PNG export of standard social media design
- AI generation: < 30s for layout generation (dominated by LLM call)

Optimizations:
- Canvas: spatial indexing (R-tree) for viewport culling; only render visible shapes
- Canvas: dirty-rect rendering; only redraw changed regions
- API: connection pooling (pgBouncer), prepared statements
- Database: partial indexes on frequently filtered columns
- JSONB: use jsonb_set for partial updates instead of full replacement
- CDN: cache exported files and asset URLs
- Redis: cache hot page content (invalidate on save)
- Worker: parallel export for multi-page files
```

**Testing**:
- `Performance: render 1,000 shapes -> maintains 60fps (measured via requestAnimationFrame timing)`
- `Performance: page load API with 500 shapes -> p95 < 200ms`
- `Performance: WebSocket cursor update round-trip -> < 50ms between two clients`
- `Load test: 50 concurrent WebSocket connections to same file -> all receive updates within 100ms`
- `Performance: export 1080x1080 PNG with 100 shapes -> completes in < 5s`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffold       --- required by everything
    |
Phase 2: Canvas Engine & Shape System        --- requires Phase 1
    |
Phase 3: File Management & Persistence       --- requires Phase 1, Phase 2
    |
Phase 4: Template System & Editor UI         --- requires Phase 3
    |
Phase 5: Brand Kit & Design System           --- requires Phase 3
    |       |-- Phase 6: Real-Time Collaboration --- requires Phase 3; can parallel Phase 5
    |       |
    |       +-- Phase 8: Export System           --- requires Phase 3; can parallel Phase 5-7
    |
Phase 7: AI Generation Engine               --- requires Phase 3, Phase 5 (for brand-aware generation)
    |
Phase 9: Accessibility Engine               --- requires Phase 2 (shapes), Phase 8 (export)
    |
Phase 10: Multi-Platform Resize & Publishing --- requires Phase 4, Phase 8
    |
Phase 11: Design-to-Code Export              --- requires Phase 5 (brand kit), Phase 8 (export)
    |
Phase 12: Enterprise Features & Hardening    --- requires all prior phases
```

**Parallelism opportunities:**
- Phases 5 and 6 can be developed concurrently after Phase 3
- Phase 8 (Export) can be developed in parallel with Phases 5-7, since it only depends on Phase 3 (page content)
- Phases 9, 10, and 11 can be developed concurrently after their dependencies are met

---

## Definition of Done (per phase)

1. All tasks implemented with the specified data structures, API signatures, and business logic.
2. All unit tests pass (Vitest).
3. All integration tests pass (Supertest for API; mocked external services).
4. ESLint passes with zero errors and zero warnings.
5. TypeScript strict mode compilation succeeds across all packages.
6. Docker build succeeds for API and worker images.
7. Database migrations run cleanly on a fresh PostgreSQL instance.
8. Feature works end-to-end as verified by at least one E2E test (Playwright where applicable).
9. New API endpoints appear in auto-generated OpenAPI spec (`@fastify/swagger`).
10. New environment variables documented in `.env.example`.
11. No security vulnerabilities introduced (OWASP API Security Top 10 considered for new endpoints).
12. WebSocket messages documented in collaboration protocol spec.
