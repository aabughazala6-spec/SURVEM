# SURVEM — PROJECT_SPEC.md

**Document:** Phase 0 Foundation Engineering Specification  
**Project:** SURVEM  
**Repository:** `aabughazala6-spec/SURVEM`  
**Phase:** Phase 0 — Foundation  
**Status:** Proposed for Approval  
**Build Strategy:** Build from Scratch  
**Engineering Standard:** Production-grade, engineering-critical, audit-ready

## 1. Purpose

Phase 0 establishes the clean technical and engineering foundation for SURVEM. It does not implement survey features yet. It establishes the application foundation, strict TypeScript environment, architecture boundaries, engineering data model foundation, Multi-Dataset local persistence, testing foundation, environment/security conventions, and reproducible verification commands.

## 2. Governing Principles

1. **Build From Scratch:** No code from the previous SurveyPro implementation may be copied into this repository. Reference applications may be used only as functional/UX references.
2. **One Platform — Multiple Clients:** Web, Mobile, and Desktop ultimately share one engineering data model, project state, and lineage.
3. **Multi-Dataset Is Mandatory:** Every project supports multiple independent datasets. Every SurveyPoint belongs to a dataset.
4. **Engineering Correctness Over Speed:** Phase 0 does not implement complex engineering calculations, but its architecture must preserve future support for deterministic calculations, units, CRS, provenance, versioning, and validation.
5. **Local Storage Is Not the Final Shared Source of Truth:** Dexie/IndexedDB is the Phase 0 local persistence layer. The long-term shared authoritative layer is the platform/cloud database; synchronization is later.
6. **No Client-Side AI Secrets:** Production AI credentials must remain server-side. AI integration is not implemented in Phase 0.

## 3. Phase 0 Scope

### In Scope

- Next.js App Router initialization
- TypeScript strict mode
- React foundation
- Tailwind foundation
- Zustand foundation
- Dexie local database
- Multi-Dataset schema
- Core domain types
- Project/dataset/task/point persistence
- Vitest configuration and initial tests
- Environment configuration
- `.env.example` and `.gitignore` protection
- Minimal application shell
- Type checking and production build
- Repository/documentation foundation

### Explicitly Out of Scope

Authentication, authorization, RLS, AI integration, rate limiting, COGO, CRS transformations, centroid, QA/QC algorithms, spatial indexing/R-tree, advanced import, DXF/DWG/KML/KMZ processing, traverse adjustment, leveling, terrain/surfaces, volumes, reporting engine, cloud synchronization, conflict resolution, production Supabase integration, billing, advanced offline synchronization, advanced dataset comparison, geotechnical modules, and geophysical modules.

## 4. Technology Baseline

- Next.js App Router 13.5.1+
- TypeScript strict
- React 18.2
- Zustand 5.x
- Dexie 4.4
- proj4 / @turf/turf as geospatial foundation
- Tailwind CSS
- Vitest
- Supabase/PostgreSQL: later phase
- NextAuth or Supabase Auth: later phase
- rbush: later P1 performance phase

## 5. Proposed Repository Structure

```text
SURVEM/
├── .github/workflows/
├── public/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── globals.css
│   ├── components/
│   │   ├── layout/
│   │   └── ui/
│   ├── db/
│   │   ├── database.ts
│   │   └── schema.ts
│   ├── domain/
│   │   ├── projects/
│   │   ├── datasets/
│   │   ├── points/
│   │   └── tasks/
│   ├── stores/
│   ├── lib/
│   │   ├── env.ts
│   │   └── utils.ts
│   └── types/
├── tests/
│   ├── db/
│   ├── domain/
│   └── fixtures/
├── docs/
├── .env.example
├── .gitignore
├── package.json
├── tsconfig.json
├── next.config.*
├── vitest.config.*
└── PROJECT_SPEC.md
```

UI components must not contain persistence logic. Domain logic must remain independently testable.

## 6. Core Domain Model

Relationship:

```text
Project 1:N Dataset
Dataset 1:N SurveyPoint
Project 1:N Task
```

A SurveyPoint MUST NOT exist without `datasetId`.

### Project

```ts
interface Project {
  id: string;
  name: string;
  description?: string;
  status: ProjectStatus;
  createdAt: Date;
  updatedAt: Date;
  ownerId: string;
}

type ProjectStatus =
  | 'planned'
  | 'active'
  | 'paused'
  | 'under_review'
  | 'approved'
  | 'completed'
  | 'archived';
```

### Dataset

```ts
interface Dataset {
  id: string;
  projectId: string;
  name: string;
  type:
    | 'topographic'
    | 'gnss'
    | 'total_station'
    | 'leveling'
    | 'control'
    | 'as_built'
    | 'imported';
  sourceFile?: string;
  sourceFormat?: string;
  status:
    | 'draft'
    | 'imported'
    | 'processing'
    | 'qa_qc'
    | 'under_review'
    | 'approved'
    | 'rejected';
  version: number;
  pointCount: number;
  createdAt: Date;
  updatedAt: Date;
  ownerId: string;
}
```

Dataset invariants:

- `projectId` is mandatory
- `id` is unique
- `version >= 1`
- `pointCount >= 0`
- `status` is controlled
- operations are dataset-scoped
- a dataset cannot silently belong to another project

### SurveyPoint

```ts
interface SurveyPoint {
  id: string;
  datasetId: string;
  pointNumber?: string;
  x?: number;
  y?: number;
  z?: number;
  code?: string;
  description?: string;
  version: number;
  sourceRecordId?: string;
  derivedFrom?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

Coordinates must later carry explicit CRS/unit context; Phase 0 does not implement transformations.

### Task

```ts
interface Task {
  id: string;
  projectId: string;
  datasetId?: string;
  title: string;
  description?: string;
  status: 'todo' | 'in_progress' | 'blocked' | 'completed';
  priority: 'low' | 'medium' | 'high';
  assignedTo?: string;
  createdAt: Date;
  updatedAt: Date;
}
```

## 7. Dexie Database

Required tables:

- `projects`
- `datasets`
- `points`
- `tasks`

Persistence must be isolated from React UI components. Database access should pass through a dedicated persistence/domain boundary.

## 8. Multi-Dataset Safety Tests

Phase 0 must prove:

1. Dataset A queries never return Dataset B points.
2. Project P1 queries never return datasets from P2.
3. Points without `datasetId` are rejected by the domain/persistence boundary.
4. Creating Dataset B does not modify Dataset A.
5. Dataset and point versions are explicitly represented.

## 9. State Management

Zustand is the client application-state foundation.

Initial state may include:

- `currentProjectId`
- `currentDatasetId`

Zustand is NOT a second database.

```text
Dexie   = persisted local engineering/project data
Zustand = transient application/session state
```

## 10. Environment and Secret Boundary

Required:

- `.env.example`
- `.env.local`

`.env.local` MUST be ignored by Git.

Only safe placeholders may exist in `.env.example`. No credentials, API keys, passwords, or tokens may be committed.

## 11. Testing Foundation

Vitest must be configured and executable.

Initial tests shall cover:

- project validation
- dataset validation
- point/dataset association
- task validation
- database initialization
- CRUD for projects/datasets/points/tasks
- dataset isolation
- basic architecture boundaries where practical

## 12. TypeScript and Build Requirements

Required:

```bash
npm run test
npm run typecheck
npm run build
```

Acceptance:

- all automated tests pass
- `tsc --noEmit` passes with 0 errors
- production build succeeds with exit code 0
- warnings are reviewed rather than blindly ignored

No `any` may be introduced merely to bypass type errors.

## 13. Package Scripts

At minimum:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

## 14. Application Shell

Phase 0 needs only a minimal working shell demonstrating:

1. Next.js runs
2. TypeScript works
3. styling works
4. application state initializes
5. Dexie initializes
6. application loads without runtime errors

No complete dashboard or feature-heavy UI is required.

## 15. Architecture Boundaries

Required dependency direction:

```text
UI
 ↓
Application / State
 ↓
Domain
 ↓
Persistence
```

UI must not directly own database transactions.

Future engineering calculations must remain independent of presentation.

Target future boundary:

```text
UI
 ↓
Application Workflow
 ↓
Engineering Domain
 ↓
Deterministic Engineering Engine
 ↓
Validated Data / Persistence
```

## 16. Future Engineering Data Requirements

Phase 0 must not block:

- explicit units
- CRS metadata
- datum context
- coordinate transformations
- observations
- control points
- geometry
- calculations
- QA/QC results
- versions
- provenance
- approvals
- deliverables

Engineering results must eventually retain their source dataset/version relationship.

## 17. Future Cloud Boundary

Phase 0 uses Dexie only for local persistence.

Future architecture:

```text
Supabase/PostgreSQL
        ↓
authoritative shared project state

Dexie/IndexedDB
        ↓
local synchronized replica
```

Synchronization is NOT implemented in Phase 0.

## 18. Security Foundation

Phase 0 requires:

- `.env.local` ignored
- no credentials in source
- no secrets in client code
- no secrets in README
- no hardcoded tokens
- declared dependencies only
- generated files reviewed before commit

Full authentication, authorization, RLS, rate limiting, server-side AI integration, and security audit belong to Phase 1.

## 19. Provenance Rules

Because SURVEM is Build-from-Scratch:

- no previous SurveyPro source code may be copied
- no old component may be transplanted
- no old engineering algorithm may be imported
- no dependency may be removed without provenance analysis
- no generated template file should be deleted without classification

## 20. Phase 0 Implementation Order

1. Repository preparation — isolated feature branch from main; do not modify main.
2. Next.js initialization — approved stack and strict TypeScript.
3. Directory foundation.
4. Environment foundation.
5. Domain types.
6. Dexie schema and persistence boundary.
7. Minimal Zustand state.
8. Vitest and initial test suites.
9. Minimal application shell.
10. Test, typecheck, build.
11. Diff, dependency, secret, architecture and quality review.

## 21. Definition of Done

### Repository
- isolated feature branch
- main untouched
- no inherited application code

### Framework
- Next.js App Router initialized
- strict TypeScript
- React and Tailwind operational

### Data
- projects, datasets, points, tasks exist
- points contain `datasetId`
- project/dataset relationships enforced
- Multi-Dataset tests pass

### State
- Zustand configured
- persistent data separated from transient state

### Testing
- Vitest configured
- domain and database tests pass

### Environment
- `.env.example` exists
- `.env.local` ignored
- no real secrets committed

### Quality
- typecheck passes with 0 errors
- tests pass
- build succeeds
- Git diff reviewed
- generated residue reviewed

### Architecture
- UI separated from persistence
- domain independently testable
- Multi-Dataset architecture present from day one
- future cloud synchronization not blocked
- future engineering engines not blocked

## 22. Foundation Gate

Phase 0 is complete only when:

1. architecture is coherent
2. data model is coherent
3. Multi-Dataset boundary is correct
4. persistence/state boundaries are correct
5. environment/security boundary is correct
6. tests exist and pass
7. TypeScript passes
8. production build passes
9. repository diff is clean
10. independent review is complete
11. human approval is granted

Only after the Foundation Gate passes may Phase 1 begin.

## 23. Final Statement

Phase 0 deliberately builds less functionality and more foundation.

Success means SURVEM has a clean, testable, traceable, Multi-Dataset engineering foundation capable of safely supporting future survey workflows and engineering engines.

“Works” is not sufficient. The Phase 0 foundation must be correct, testable, traceable, secure, reviewable, and releasable.
