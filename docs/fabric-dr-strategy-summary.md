# Fabric Disaster Recovery Strategy Summary

> This document is a proposed DR design derived from patterns and building blocks already present in this repository. It is intentionally opinionated, so teams can adapt it to their own RPO/RTO targets, security model, and governance requirements.

## 1. Purpose

This summary turns the repo's existing BCDR, backup/recovery, CI/CD, monitoring, and management assets into a single operating model for disaster recovery (DR) across:

- **Capacities**
- **Workspaces**
- **Git-backed Fabric items**
- **Lakehouse and warehouse data**
- **Security and operational recovery**

The repo already shows several key building blocks:

- The **BCDR accelerator** captures primary-region metadata, then recreates workspaces, reconnects Git, restores items, and recovers lakehouse/warehouse data in a DR region.
- The **data warehouse recovery playbook** separates backup of workspace config/security, warehouse metadata, warehouse data, and warehouse security.
- The **CI/CD accelerators** provide repeatable workspace creation, branch isolation, item rebinding, and promotion across DEV/UAT/PROD.
- The **platform monitoring solution** provides capacity, inventory, audit, gateway, and activity observability that can be repurposed for DR readiness and post-failover validation.
- The **MicrosoftFabricMgmt** module provides automation primitives for creating workspaces, assigning capacities, restoring deleted workspaces, and managing RBAC.

## 2. Design principles

1. **Treat capacity and workspace recovery as separate problems.**
   Capacity DR is about regional placement and OneLake geo-redundancy. Workspace DR is about recreating definitions, bindings, permissions, and operational settings.
2. **Git is the control plane for supported items.**
   If an item can be represented in source control, recovery should prefer rehydration from Git over ad hoc recreation.
3. **OneLake replication is necessary but not sufficient.**
   Capacity-level DR protects replicated data, but the repo makes clear that items still need to be recovered into new workspaces in the secondary region.
4. **Security and configuration must be backed up outside the workspace.**
   Workspace permissions, warehouse permissions, connection settings, and other environment-specific settings must be scripted and retained independently.
5. **Observability is part of DR, not an afterthought.**
   Inventory snapshots, audit trails, and capacity telemetry are required to prove the environment is ready and to validate recovery.
6. **Use tiers.**
   Not every workload needs the same DR investment. This document therefore defines a **minimal DR** pattern and a **maximum DR** pattern.

## 3. Recovery domains

### 3.1 Capacity domain

A Fabric capacity is the top-level DR boundary for OneLake geo-redundancy. If the DR setting is enabled on the capacity, OneLake data for workspaces assigned to that capacity is replicated to another region. Because of that, capacity assignment should be treated as a first-class DR design decision.

### 3.2 Workspace domain

Workspaces carry item definitions, RBAC, connections, deployment topology, and operational ownership. The repo's BCDR content shows that a successful DR process needs enough metadata to recreate workspaces, reconnect them to Git, restore supported items, and then reapply workspace roles and data bindings.

### 3.3 Data domain

The warehouse recovery playbook separates:

- **Warehouse metadata**
- **Warehouse data in OneLake/delta-parquet**
- **Workspace security**
- **Warehouse security**

That separation should also be used for broader Fabric DR planning.

### 3.4 Operations domain

Monitoring, activity logs, scanner snapshots, gateway telemetry, and inventory extraction form the evidence set for:

- readiness reviews
- dependency mapping
- failover triage
- post-recovery validation

## 4. Proposed target operating model

I recommend organizing Fabric into four DR-oriented workspace classes.

### 4.1 Control workspace

A dedicated control workspace stores DR metadata, recovery notebooks, automation logs, and validation outputs.

**Purpose**
- Run the primary-region metadata capture process.
- Store recovery state and generated backup artifacts.
- Host DR orchestration notebooks/pipelines.

**Contents**
- BCDR metadata tables
- recovery runbooks
- generated permission scripts
- environment inventory extracts
- post-failover validation reports

### 4.2 Application workspaces

These are the business workspaces containing lakehouses, warehouses, notebooks, pipelines, semantic models, and reports.

**Purpose**
- Host the actual data and analytics workloads.
- Be recoverable from Git plus replicated OneLake data plus scripted security/configuration.

### 4.3 Shared services workspace

A workspace for centrally reused assets such as platform monitoring, gateway reporting, admin notebooks, and operational dashboards.

**Purpose**
- Preserve tenant-level observability during and after DR.
- Support rapid assessment of capacity health, failed jobs, gateway state, and workspace inventory.

### 4.4 DR staging workspace(s)

One or more temporary or warm workspaces in the secondary region for rehydration, validation, and cutover.

**Purpose**
- Recreate recovered items.
- Restore or attach replicated data.
- Perform smoke tests before business cutover.

## 5. Capacity strategy across workspaces

The most important design decision is how workspaces map to capacities.

### Option A: Consolidated capacity model

Many workspaces share one primary capacity, with one secondary capacity created only during DR.

**Pros**
- Lowest steady-state cost
- Easiest to administer
- Fits smaller environments

**Cons**
- Larger blast radius
- Harder to prioritize workloads during recovery
- More contention during rehydration and testing

### Option B: Tiered capacity model

Critical workspaces are grouped onto one or more dedicated capacities, while less critical workspaces share another capacity.

**Pros**
- Better isolation
- Easier business prioritization
- More predictable recovery sequencing

**Cons**
- Higher cost
- More capacity planning effort

### Option C: Paired primary/secondary capacity model

Critical workloads have a preplanned secondary-region target capacity and a repeatable mapping from source workspace to target capacity.

**Pros**
- Fastest recovery
- Clearest runbooks
- Best for regulated or revenue-critical platforms

**Cons**
- Highest cost and governance overhead

## 6. Minimal DR strategy

This is the **cost-aware** pattern. It aims for recoverability with moderate RTO and moderate RPO, using mostly cold or pilot-light capabilities.

### 6.1 Where it fits

Use minimal DR when:
- the platform can tolerate hours to a day of recovery work
- not all workspaces are mission-critical
- budget is limited
- the team can accept more manual cutover steps

### 6.2 Capacity pattern

- Keep production workspaces on their normal primary capacity.
- Enable capacity-level DR/geo-redundancy for capacities that host critical lakehouse and warehouse data.
- Do **not** keep an always-running full-size secondary capacity for every environment.
- Instead, maintain a documented secondary-region target and provision recovery capacity when DR is declared, following the warehouse playbook pattern.

### 6.3 Workspace pattern

- Connect all supported production items to Azure DevOps-backed Git.
- Store source of truth for supported items in Git.
- Schedule metadata capture from the BCDR accelerator to a dedicated control workspace.
- Script workspace permissions and warehouse permissions daily and store them outside the affected workspace.
- Use CI/CD templates to standardize workspace names, branches, and post-deployment rebinding.

### 6.4 Data pattern

- Rely on capacity-level OneLake replication for lakehouse and warehouse data that sits behind the BCDR feature.
- For warehouses, separately preserve metadata through Git/source control and preserve security through generated SQL scripts.
- For non-Git-backed or unsupported items, maintain a manual recovery appendix with named owners.

### 6.5 Recovery workflow

1. Declare DR and identify the impacted primary capacity/workspaces.
2. Provision a new capacity in the secondary region.
3. Create DR target workspaces.
4. Restore workspace definitions from captured metadata.
5. Reconnect workspaces to Git and sync the correct commit.
6. Restore lakehouse/warehouse item structures.
7. Reapply workspace RBAC and warehouse permissions from generated scripts.
8. Rebind connections, default lakehouses, pipelines, semantic models, and reports.
9. Validate using platform monitoring dashboards, inventory extracts, and smoke tests.
10. Update client connection strings/endpoints where required.

### 6.6 Expected trade-offs

- **RPO:** good for replicated OneLake data; weaker for manually captured config if backup frequency is low
- **RTO:** medium, because new capacity creation and manual rebinding still take time
- **Operational burden:** moderate
- **Cost:** low to medium

## 7. Maximum DR strategy

This is the **resilience-first** pattern. It is designed for low RTO, stronger validation, and clearer failover execution.

### 7.1 Where it fits

Use maximum DR when:
- analytics downtime is materially expensive
- there are contractual or regulatory recovery commitments
- multiple business-critical domains depend on Fabric
- the organization can fund warm standby capabilities

### 7.2 Capacity pattern

- Segment workspaces by business criticality across dedicated capacities.
- Pre-define a paired secondary-region capacity strategy for each critical production capacity.
- Keep secondary capacities pre-created and pre-sized for the critical recovery tier.
- Reserve sufficient headroom for parallel rehydration, validation, and user cutover.

### 7.3 Workspace pattern

- Maintain a control workspace in the primary region and a mirrored DR-control workspace plan in the secondary region.
- Keep all supported artifacts in Git and enforce promotion through standardized CI/CD.
- Use deployment pipelines for structured DEV/TEST/PROD promotion where applicable.
- Maintain pre-created or template-driven DR workspaces for the highest tier so recovery becomes resynchronization rather than net-new creation.
- Use automated management tooling to reapply RBAC, capacity assignment, identities, domains, and workspace settings.

### 7.4 Data pattern

- Enable OneLake geo-redundancy on every critical capacity.
- Treat warehouse metadata, warehouse data, workspace config, and warehouse security as separate protected assets.
- Schedule daily or more frequent exports of workspace permissions, warehouse permissions, and environment-specific configuration.
- Maintain dependency maps for pipelines, shortcuts, connections, gateways, semantic models, notebooks, and reports.
- For critical downstream consumers, pre-document alternate connection endpoints and rollback steps.

### 7.5 Monitoring and validation pattern

- Run platform monitoring continuously for capacity, gateway, audit, and inventory data.
- Use scanner/inventory outputs to compare primary and DR states.
- Build a DR scorecard that validates:
  - workspace exists
  - workspace assigned to intended capacity
  - required items recreated
  - Git commit aligned
  - permissions restored
  - gateway and connection dependencies healthy
  - smoke-test notebooks and pipelines succeeded

### 7.6 Recovery workflow

1. Trigger the DR orchestration plan for the affected workload tier.
2. Activate or scale the paired secondary capacity if needed.
3. Recover or synchronize target workspaces from templates/metadata.
4. Reconnect to Git and sync to the approved recovery commit.
5. Restore or attach replicated data.
6. Execute automated post-activity steps for rebinding and connection swaps.
7. Reapply security and environment settings using scripted automation.
8. Run validation notebooks, deployment checks, and business smoke tests.
9. Switch clients, schedules, and support processes to the DR environment.
10. Monitor the recovered estate continuously until the primary region is either rebuilt or formally failed back.

### 7.7 Expected trade-offs

- **RPO:** strongest of the two patterns
- **RTO:** lowest practical RTO based on the repo building blocks
- **Operational burden:** high, but mostly shifted into automation and rehearsals
- **Cost:** highest

## 8. Minimal vs maximum DR at a glance

| Area | Minimal DR | Maximum DR |
|---|---|---|
| Secondary capacity | Provision on demand | Pre-created / warm standby |
| Workspace recreation | Metadata-driven, mostly on declaration | Template-driven plus metadata sync |
| Git usage | Required for critical supported items | Required for all supported items |
| Security backup | Daily scripts | Daily or more frequent scripted exports |
| Monitoring | Used for recovery validation | Used for readiness, detection, validation, and continuous assurance |
| Rebinding and post-deploy tasks | Partly manual | Automated wherever possible |
| Best fit | Cost-sensitive environments | Mission-critical analytics platforms |

## 9. Recommended workspace tiers

I recommend assigning each workspace to one of these recovery tiers.

### Tier 0 - Control / DR operations

**Examples**
- DR control workspace
- platform monitoring workspace
- admin automation workspace

**Recommendation**
- Use the **maximum DR** pattern.
- Without these, coordinated recovery becomes much harder.

### Tier 1 - Business-critical production analytics

**Examples**
- executive reporting
- financial analytics
- customer-facing operational analytics
- regulated reporting

**Recommendation**
- Prefer **maximum DR**.
- Use dedicated or strongly isolated capacities.

### Tier 2 - Important but not mission-critical

**Examples**
- departmental reporting
- advanced analytics sandboxes with reusable data products

**Recommendation**
- Use **minimal DR** by default.
- Upgrade to maximum DR only where business impact justifies it.

### Tier 3 - Development / experimentation

**Examples**
- feature branches
- temporary project workspaces
- personal or lab environments

**Recommendation**
- Usually no formal DR beyond Git and standard CI/CD reproducibility.

## 10. Governance recommendations

### 10.1 Naming and mapping

Maintain a DR registry with:
- source workspace name and ID
- source capacity name and ID
- target DR workspace naming convention
- target DR capacity mapping
- Git repository, branch, and expected path
- owner, deputy owner, and approver
- recovery tier

### 10.2 Backup schedule

At minimum:
- workspace metadata capture: multiple times per day for critical workloads
- workspace permission export: daily
- warehouse permission export: daily
- inventory/scanner snapshots: at least every 30 to 120 minutes using the repo monitoring pattern
- validation smoke tests: daily for Tier 0/1, weekly for Tier 2

### 10.3 Testing cadence

- Tier 0/1: quarterly DR simulation
- Tier 2: semiannual DR simulation
- Tier 3: no dedicated DR test; rely on rebuild automation

Each rehearsal should measure:
- time to provision capacity
- time to recreate workspaces
- time to sync Git
- time to restore permissions
- time to validate consumer access
- issues caused by unsupported items or manual dependencies

## 11. Suggested implementation roadmap

### Phase 1 - Establish recoverability baseline

- Put all supported production items into Git.
- Deploy the BCDR metadata capture process.
- Deploy the platform monitoring solution for capacity, inventory, and audit evidence.
- Script workspace and warehouse permissions.
- Document unsupported/manual steps.

### Phase 2 - Standardize environment rebuild

- Use CI/CD patterns from this repo to standardize workspace creation and rebinding.
- Define capacity-to-workspace recovery mappings.
- Create validation notebooks and smoke tests.
- Store all DR outputs in the control workspace and an external repository.

### Phase 3 - Add fast recovery for critical tiers

- Pre-create paired secondary capacities.
- Template DR workspaces for Tier 0/1 domains.
- Automate post-recovery rebinding and connection swaps.
- Run quarterly rehearsals and refine RTO/RPO targets.

## 12. Final recommendation

If you want a single default posture derived from this repo, my recommendation is:

- Use **maximum DR** for the **control/monitoring layer** and for **Tier 1 production workspaces**.
- Use **minimal DR** for **Tier 2 production workspaces**.
- Use **rebuild-only** discipline for **Tier 3 development workspaces**.

That blended model best matches the repository's strengths:
- metadata-driven recovery from the BCDR accelerator
- source-controlled item recreation through Git and CI/CD
- separate backup of permissions and warehouse security
- platform-wide observability through monitoring
- scripted workspace/capacity automation through the management module

In short: **replicate data at the capacity layer, reconstruct items at the workspace layer, preserve security outside the workspace, and validate everything with continuous monitoring.**
