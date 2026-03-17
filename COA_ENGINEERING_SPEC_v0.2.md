# Baku Course of Action (COA) — Engineering Specification

**Version:** 0.2 (derived from Product Spec Working Draft)  
**Date:** 2026-03-16  
**Status:** Engineering Draft  
**Audience:** Product, Engineering, Design, Data, Integrations

---

## 1) Purpose and Scope

This document translates the COA product specification into implementable engineering requirements for V1 delivery.

COA must deliver:
- A **Deal Tile** (static fact object).
- A **Situation** (dynamic signal object with evidence).
- **COA Paths** (decision graph with stage-aware pruning, move states, and approvals).
- A **Move Inventory** view by category and move state.
- A **mobile-first** interaction model with tablet/desktop graph mode.

Out of scope for V1 unless otherwise noted:
- Fully autonomous AI path generation.
- Enterprise-grade offline write queue conflict resolution.
- Full portfolio analytics beyond manager intervention baseline.

---

## 2) Canonical Domain Model

### 2.1 Core Entities

#### Deal
- `id` (UUID)
- `companyName` (string, required)
- `dealValue` (integer cents, required)
- `currency` (ISO 4217, required)
- `products` (array<string>, required, min 1)
- `companyPoc` (object, required)
  - `name` (string)
  - `title` (string)
- `estimatedCloseDate` (date, required)
- `assignedRep` (object, required)
  - `userId` (UUID)
  - `displayName` (string)
  - `avatarUrl` (string | null)
- `coreProblem` (string, required, max 120 chars)
- `stage` (enum: `DISCOVER|QUALIFY|ADVANCE|PROPOSE|NEGOTIATE|CLOSE`, required)
- `crmExternalId` (string | null)
- `createdAt`, `updatedAt` (timestamps)

Validation:
- Reject writes when `coreProblem.length > 120`.
- Compute `daysToClose = estimatedCloseDate - today`.
- Derive urgency label for UI: `GREEN (>30)`, `AMBER (15..30)`, `RED (<15)`.

#### SituationSignal
- `id` (UUID)
- `dealId` (UUID, FK)
- `type` (enum: `RISK|GAP|OK|INFO`)
- `title` (string, required, <= 120)
- `detail` (string, required, <= 280)
- `sourceType` (enum: `CALL|EMAIL|CRM|MANUAL|SYSTEM`)
- `sourceRef` (string, required; call clip id/email message id/CRM event id/manual note id)
- `sourceUrl` (string | null; deep link for evidence)
- `origin` (enum: `INTELLIGENCE|REP|MANAGER`)
- `status` (enum: `ACTIVE|DISMISSED|ESCALATED`)
- `managerOverride` (boolean)
- `createdByUserId` (UUID | null)
- `createdAt`, `updatedAt`

Permissions:
- Rep can create `origin=REP` signals.
- Rep cannot dismiss/edit `origin=INTELLIGENCE` signals.
- Manager can edit/dismiss/escalate any signal.

#### DecisionNode
- `id` (UUID)
- `graphVersionId` (UUID)
- `columnIndex` (0..6)
- `stage` (enum as Deal stage)
- `type` (enum: `ROOT|ENTRY|QUALIFY|ADVANCE|PROPOSE|NEGOTIATE|CLOSE`)
- `title` (string)
- `subtitle` (string)
- `ifStatement` (string)
- `thenStatement` (string)
- `isRoot` (boolean)
- `sortOrder` (integer)

#### DecisionEdge
- `id` (UUID)
- `graphVersionId` (UUID)
- `fromNodeId` (UUID)
- `toNodeId` (UUID)
- `defaultState` (enum: `ACTIVE|PRUNED|LAST_CHANCE`)

#### CoaPath
- `id` (UUID)
- `dealId` (UUID)
- `nodeIds` (array<UUID>)
- `title` (string)
- `relevanceScore` (0..100 int)
- `logicSummary` (derived from IF/THEN)
- `state` (enum: `ACTIVE|PRUNED|ACCEPTED`)
- `pruneReason` (string | null)
- `calculatedAt` (timestamp)

#### PathStep
- `id` (UUID)
- `pathId` (UUID)
- `nodeId` (UUID)
- `index` (integer)
- `state` (enum: `LOCKED|INTERACTIVE|SKIPPED|ACCEPTED|PRUNED`)

#### MoveDefinition
- `id` (UUID)
- `slug` (string unique)
- `name` (string)
- `category` (enum: `INTEL|THREADING|ADVANCING|CLOSE`)
- `channel` (enum: `CALL|EMAIL|TEXT|LINKEDIN|EXEC_TO_EXEC|CO_CREATE|BACK_CHANNEL`)
- `whatTemplate` (string)
- `whyTemplate` (string)
- `applicableStages` (array<Deal.stage>)

#### StepMoveInstance
- `id` (UUID)
- `stepId` (UUID)
- `moveDefinitionId` (UUID)
- `whatText` (resolved text)
- `whyText` (resolved text)
- `state` (enum: `AVAILABLE|LAST_CHANCE|OFF_TABLE`)
- `selected` (boolean)
- `interactive` (boolean)

#### PlanVersion
- `id` (UUID)
- `dealId` (UUID)
- `versionNumber` (integer)
- `status` (enum: `DRAFT|COMMITTED`)
- `committedAt` (timestamp | null)
- `committedByUserId` (UUID | null)
- `winProbabilityDeltaBps` (integer, optional)

#### PlanDecisionEvent (audit log)
- `id` (UUID)
- `planVersionId` (UUID)
- `dealId` (UUID)
- `nodeId` (UUID)
- `stepId` (UUID)
- `eventType` (enum: `MOVE_SELECTED|MOVE_APPROVED|STEP_SKIPPED|MOVE_OVERRIDDEN|PATH_ASSIGNED|SIGNAL_DISMISSED|PLAN_COMMITTED`)
- `payload` (JSON)
- `actorUserId` (UUID)
- `actorRole` (enum: `REP|MANAGER|EXEC|SYSTEM`)
- `createdAt` (timestamp)

---

## 3) State Machines

### 3.1 Move State
`AVAILABLE -> LAST_CHANCE -> OFF_TABLE`
- Transition to `LAST_CHANCE` when current stage is final stage in `applicableStages`.
- Transition to `OFF_TABLE` when deal advances beyond applicability or path/node is pruned.
- `OFF_TABLE` remains visible but non-interactive.

### 3.2 Node State
- `ROOT`: neutral, no actions.
- `AVAILABLE`: node active and has interactive step.
- `LAST_CHANCE`: active but closing with current stage boundary.
- `PRUNED`: non-interactive, low opacity, reason shown.
- `ACCEPTED`: committed move in node.

### 3.3 Step Progression
- First step in active path is `INTERACTIVE`; others `LOCKED`.
- On Approve: step -> `ACCEPTED`, next step -> `INTERACTIVE`.
- On Skip: step -> `SKIPPED`, next step -> `INTERACTIVE`.
- If path pruned: remaining steps -> `PRUNED`.

### 3.4 Plan Versioning
- Any active editing occurs on latest `DRAFT`.
- `Approve Plan` sets `DRAFT -> COMMITTED`, immutable thereafter.
- Subsequent edits create new `DRAFT` with `versionNumber+1`.

---

## 4) Decision Engine Requirements

### 4.1 Inputs
- Deal facts (Deal Tile fields)
- Active Situation signals
- Static graph definition (40+ nodes, edges, stage mapping)
- Move definitions + eligibility rules

### 4.2 Outputs
- Active/pruned nodes and edges
- CoA paths with `relevanceScore`
- Per-step move instances with state (`AVAILABLE|LAST_CHANCE|OFF_TABLE`)
- Explanation strings for each prune decision

### 4.3 Recalculation Triggers
- Situation signal create/update/dismiss
- Deal stage change
- Core field updates (`closeDate`, `value`, `products`, `coreProblem`)
- Manager path assignment / override

### 4.4 Non-Functional
- P95 recomputation under 500ms per deal.
- Deterministic output for same input set (idempotent evaluation).
- Event-sourced traceability for each engine run (`inputsHash`, `rulesVersion`, `runId`).

---

## 5) API Surface (V1)

### 5.1 Deal Tile
- `GET /deals/:dealId`
- `PATCH /deals/:dealId` (validates `coreProblem <= 120`)

### 5.2 Situation
- `GET /deals/:dealId/situation?status=ACTIVE`
- `POST /deals/:dealId/situation` (rep/manager)
- `PATCH /deals/:dealId/situation/:signalId` (manager for intelligence-origin edits)
- `POST /deals/:dealId/situation/:signalId/dismiss` (manager)
- `POST /deals/:dealId/situation/:signalId/escalate` (manager)

### 5.3 COA Graph + Paths
- `GET /deals/:dealId/coa` returns:
  - graph nodes/edges + computed visual states
  - paths + relevance + IF/THEN
  - steps + move instances
  - execution plan summary

### 5.4 Move Interactions
- `POST /deals/:dealId/steps/:stepId/select-move`
- `POST /deals/:dealId/steps/:stepId/approve`
- `POST /deals/:dealId/steps/:stepId/skip`

### 5.5 Plan Commit
- `POST /deals/:dealId/plans/commit`
  - side-effects:
    - emit Slack event (`plan.committed`)
    - enqueue CRM writeback (`opportunity.next_steps.updated`)
    - persist immutable `PlanVersion`

### 5.6 Manager Interventions
- `POST /deals/:dealId/nodes/:nodeId/comment`
- `POST /deals/:dealId/nodes/:nodeId/assign-path`
- `POST /deals/:dealId/overrides/move`

---

## 6) UI/Frontend Engineering Requirements

### 6.1 Mobile (primary target: 375px)
- Single-column layout.
- Deal Tile first; Situation second; Decision Tree third.
- Sticky footer with committed moves + plan stats.
- Minimum tap target 44px.
- Pinch zoom enabled (`viewport` scale 0.5 to 4).

### 6.2 Decision Tree Rendering
- Mobile: vertical step-chain rendering.
- Tablet/Desktop: horizontal SVG canvas with Bezier edges.
- Edge styles:
  - Active: solid blue 1.5px
  - Accepted: solid green 2px
  - Last chance: dashed amber 1px
  - Pruned: dashed grey 1px @ 40% opacity

### 6.3 Node Card Behavior
- Header always visible (type/title/subtitle/state).
- Tap expands body with IF/THEN + move pills + Approve/Skip controls.
- Off-table moves visible struck-through and disabled.

### 6.4 Accessibility
- WCAG AA contrast for all state colors.
- Screen-reader labels for node/move states.
- Keyboard interaction support for desktop mode.

---

## 7) Integrations and Side Effects

### 7.1 CRM Sync (phase 1)
- Read mapping from Salesforce/HubSpot into Deal fields.
- Writeback only on plan commit:
  - next steps summary
  - selected moves list
  - commit timestamp

### 7.2 Intelligence Ingestion
- Ingest events from call transcript and email processors.
- Normalize to SituationSignal events.
- Maintain evidence link (`sourceUrl`/`sourceRef`) mandatory for `INTELLIGENCE` origin.

### 7.3 Slack Notifications
- On plan commit, send manager summary:
  - Deal, rep, stage, approved moves, skipped steps, timestamp.

---

## 8) Role-Based Access Control

- **Rep**
  - View all deal objects in owned deals.
  - Add manual signals.
  - Select/approve/skip moves.
  - Commit plan.
  - Cannot dismiss intelligence-origin signals.

- **Manager**
  - All rep permissions across managed team deals.
  - Edit/dismiss/escalate signals.
  - Assign paths, comment, override moves.

- **Exec**
  - Read portfolio-level COA.
  - Intervention actions equivalent to manager (configurable).

---

## 9) Observability, Logging, and Analytics

- Structured logs for every decision engine run.
- Audit stream for move approvals, skips, overrides, plan commits.
- Metrics:
  - Time from stage entry to first approved move.
  - % last-chance moves used vs expired.
  - Off-table move count by category at each stage.
  - Manager override frequency.

---

## 10) Performance Targets

- Deal COA view API P95 < 700ms server time.
- Graph render interaction (expand node) < 100ms on modern iPhone.
- Initial mobile payload target < 400KB gzipped (excluding avatars).

---

## 11) Delivery Plan

### Milestone 1 — Core UX/Workflow (ship-first)
- Deal Tile + Situation + COA tree + move states + approve/skip + plan commit.
- Manager signal edit/dismiss.
- Mobile-first UI.

### Milestone 2 — Integrations
- CRM read/write mapping.
- Slack notifications.
- Intelligence ingestion pipeline.

### Milestone 3 — Orchestration Views
- Team manager view.
- Exec portfolio view.
- Historical plan version browser.

---

## 12) Open Engineering Questions

1. Path generation strategy in V1: rules-only vs rules+ML reranking.
2. Conflict model for concurrent rep/manager edits on same node.
3. Offline mode policy for approve/skip actions.
4. Degree of CRM writeback granularity per move.
5. Rules version rollout and rollback strategy.

---

## 13) Naming Contract (must be enforced in code/docs/UI copy)

Use only these canonical terms in DTOs/UI labels/docs:
- Deal Tile
- Situation
- Signal (`RISK|GAP|OK|INFO`)
- COA path
- Step
- Move
- Move state (`Available|Last Chance|Off the Table`)
- Approve
- Skip
- Execution Plan
- Pruned
- Decision tree
- Move Inventory

Disallowed synonyms (lint in copy where feasible):
- `playbook` for COA path
- `workflow` for COA path
- `status`/`flag` for signal type
- `active/warning/disabled` for move state labels
- `accept/select/confirm` for Approve
- `dismiss/pass` for Skip

---

## 14) Acceptance Criteria (V1)

1. User can open a deal and see mandatory Deal Tile fields with correct close-date urgency color.
2. User sees Situation signals with typed color stripe and evidence links.
3. Decision tree responds to stage and signal changes with visible pruning.
4. User can approve or skip step actions; state transitions are persisted and auditable.
5. Move inventory displays category grouping and current move states.
6. Rep permissions are enforced; manager overrides are logged.
7. Plan commit creates immutable version, sends Slack event, and queues CRM writeback.
8. Mobile layout is functional at 375px with touch target and sticky footer requirements met.
