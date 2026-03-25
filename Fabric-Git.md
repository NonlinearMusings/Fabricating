# CI/CD with Fabric and GitHub

## Overview

Microsoft Fabric provides two complementary components for CI/CD:
- **Git Integration** (CI): Workspace syncs with GitHub repository for version control and collaboration
- **Deployment Pipelines** (CD): Automates promotion between Dev/Test/Prod workspace stages

You can use these together or separately depending on your workflow needs.

### Understanding the Architecture: Git Branches vs. Deployment Pipelines

Before setup, understand these are separate concepts:

- **Git Branches** (in GitHub): Where individual developers integrate their work via PRs and merges
- **Deployment Pipeline Stages** (in Fabric): Workspace tiers (Dev/Test/Prod) that promote entire workspaces

**One shared Dev integration workspace connected to `main` branch** → receives all merged developer work → promotes to Test/Prod via pipeline.

Developers can either:
- Work directly against this shared integration workspace (smaller teams)
- Work in personal feature workspaces and merge through PRs (larger teams, safer isolation)

```
Developer #1              Developer #2
(feature/report)         (feature/model)
    ↓ PR to main            ↓ PR to main
    ↓ (review & merge)      ↓ (review & merge)
    
    Shared Dev Workspace (main branch) ← pulls all merged changes
    ↓
Deployment Pipeline: Dev → Test → Prod
```

---

## Option #1: Fabric Workspace-Based Development

**Option #1** works directly in Fabric workspaces within the browser—ideal for analysts, report creators, and teams unfamiliar with local IDE tools.

### Workspace Decision Tree (Choose Your Team Model)

Use this decision tree first, then follow the matching workflow sections below.

1. Do developers need isolated Fabric workspaces while building features?
   - **No**: Use **Model A (Shared Dev Workspace)**
   - **Yes**: Use **Model B (Personal Dev Workspaces + Shared Integration Workspace)**

2. In both models, use the same promotion rule:
   - **Deployment pipelines promote the shared stage workspace only**
   - Never promote individual feature branches or personal developer workspaces directly

Model summary:
- **Model A: Shared Dev Workspace**
  - One Dev workspace connected to `main`
  - Each developer works through their own feature branch + PR, then shared Dev pulls from `main`
  - This model is most practical when feature-branch work happens primarily in VS Code/local Git, or when use of the shared Fabric workspace is tightly coordinated
- **Model B: Personal Dev Workspaces + Shared Integration Workspace** (recommended for larger teams)
  - One shared Dev integration workspace connected to `main`
  - Each developer has a personal feature workspace connected to a feature branch
  - Feature branches merge to `main`, shared Dev pulls updates, then promotes to Test/Prod

### Step 1: Initial Setup (One-Time)

1. **Create three workspaces** (requires F2+ capacity):
   - `MyProject-Dev` (shared integration workspace - connects to `main` branch)
   - `MyProject-Test` (test workspace)
   - `MyProject-Prod` (production workspace)

   Optional for per-developer isolation:
   - `MyProject-Dev-<username>` workspaces (one per developer, usually connected to feature branches)

2. **Connect Dev workspace to GitHub (must be `main` branch)**:
   - Go to workspace settings → **Git integration**
   - Select GitHub provider
   - Authenticate with GitHub (requires personal access token with `contents: read/write`)
   - Specify: Repository URL, **branch = `main`**, and folder (e.g., `/fabric-items`)
   - Select **Connect and sync**
   - ⚠️ **Important**: This shared Dev workspace is the source of truth. All developer branches merge to `main` here.

3. **Create a deployment pipeline**:
   - From Workspaces flyout, select **Deployment pipelines** → **Create pipeline**
   - Name it (e.g., `MyProject Pipeline`)
   - Define stages: **Development**, **Test**, **Production**
   - Assign workspaces: Dev (main) → Development stage, Test → Test stage, Prod → Production stage

### Step 2: Multi-Developer Feature Workflow (How work gets integrated)

Follow the model selected in the decision tree.

#### Model A Workflow: Shared Workspace Development (simple)

1. Developers create feature branches in GitHub.
2. Developers commit from the shared Dev workspace by using **Commit to new branch** / **Switch branch**, or commit locally through VS Code.
3. Developers open PRs from feature branches to `main`.
4. After merge, the shared Dev workspace updates from `main`.
5. Deployments to Test/Prod always use the shared Dev workspace.

This is simpler but has more contention because everyone edits in one workspace.

#### Model B Workflow: Personal Workspace Per Developer (recommended)

**For each developer working on a new feature:**

1. **Create an isolated feature branch and workspace** (in Fabric):
   - Go to Dev workspace → Source control → **Branches** tab
   - Click **Branch out to another workspace**
   - Select **Create new workspace**
   - Name it: `MyProject-Dev-feature-<username>` (e.g., `MyProject-Dev-feature-alice`)
   - Name branch: `feature/my-feature` (e.g., `feature/sales-dashboard`)
   - This creates a new branch in Git AND a new isolated workspace

2. **Work in the feature workspace** (isolated from other developers):
   - Develop your items (notebooks, models, reports)
   - Commit frequently to your feature branch: **Source control** → **Commits** → select items → **Commit**
   - Your changes stay in your feature branch (`feature/sales-dashboard`)
   - Other developers don't see your uncommitted changes

3. **Create a Pull Request on GitHub**:
   - In GitHub, create PR: `feature/sales-dashboard` → `main`
   - Add description of changes
   - Assign reviewers from your team
   
4. **Code Review and Approval** (by your team):
   - Team reviews your changes
   - Approves or requests changes
   - Once approved, **merge PR to main** on GitHub
   - Your commits are now integrated into the `main` branch

5. **Clean up the feature workspace** (optional but recommended):
   - After merge, you can delete the temporary feature workspace (no longer needed)
   - Reconnect to the shared Dev workspace

### Step 3: Keeping the Shared Dev Workspace Current (After merges)

After developers merge their PRs to `main`, the shared Dev workspace needs to pull those changes:

1. **In the shared `MyProject-Dev` workspace**, click **Source control** icon
2. Go to **Updates** tab
3. You'll see notifications that `main` has new changes
4. Click **Update all**
5. The workspace syncs to the latest `main` branch, pulling in all merged developer work

This is the **critical step** that consolidates all developer branches into the deployment source.

Applies to both models:
- Model A: Pulls merged Git changes into the same shared workspace developers used.
- Model B: Pulls merged changes from personal feature workspaces into the shared integration workspace.

### Step 4: Promotion from Dev → Test (With all developer work)

Once the shared Dev workspace is current (has all merged PRs):

1. **In Fabric deployment pipeline**:
   - Open the pipeline
   - Navigate to **Development** stage
   - Click **Deploy to Test** button
   - Review the items to deploy (should include all latest merged features)
   - Select **Full deployment** (all items since last deployment)
   - Click **Deploy**

2. **Test workspace receives everything** that was merged to `main`

Promotion rule for both models:
- You do not promote personal developer workspaces.
- You promote the shared stage workspace assigned to the Development stage.

### Step 5: Promote Test → Prod (Manual)

1. **In Fabric deployment pipeline**:
   - Navigate to **Test** stage
   - Click **Deploy to Production**
   - Optionally use **deployment rules** to override environment-specific settings (e.g., database connection ID, lakehouse ID)
   - Click **Deploy**

2. **Production is now live** with the tested changes

---

## Option #2: VS Code-Based Development

**Option #2** uses VS Code with Power BI projects (.pbip files) or edits item definitions directly—ideal for teams that prefer full code-first workflows and Git branch management.

### Workspace Decision Tree (Choose Your Team Model)

Same as Option #1 decision tree:
- **Shared Dev workspace** always connected to `main` branch
- Individual developers work in **feature branches**
- Work integrates via **Git PRs and merges**
- Deployment pipelines promote the **shared Dev workspace** (not individual branches)

Mapping to models:
- **Model A**: VS Code users branch and PR to `main`, then shared Dev updates from Git
- **Model B**: VS Code users branch and PR from personal branches/workspaces, then shared Dev updates from Git

### Step 1: Initial Setup (One-Time)

1. **Create same three workspaces** as Option #1

2. **Connect Dev workspace to GitHub (main branch)**:
   - Same as Option #1 - ensure you're on the `main` branch
   - This is the shared development workspace

3. **Create deployment pipeline** (same as Option #1)

### Step 2: Feature Branch Development (Recommended workflow)

1. **Clone the repository locally**:
   ```bash
   git clone <repository-url>
   cd fabric-project
   ```

2. **Create a feature branch** (for your work):
   ```bash
   git checkout -b feature/my-feature
   ```

3. **Download/edit items from Dev workspace** (optional):
   - In Fabric, right-click item → **Download** → get the `.pbip` or definition files
   - Or fetch definitions from your Git repo
   - Edit in VS Code

4. **Make changes and commit locally**:
   ```bash
   # Edit item definitions (`.notebookmd`, `.json`, `.pbip` files, etc.)
   git add .
   git commit -m "Add new dashboard and fact table"
   ```

5. **Push your feature branch to GitHub**:
   ```bash
   git push -u origin feature/my-feature
   ```

6. **Create a Pull Request on GitHub**:
   - Go to GitHub → create PR from `feature/my-feature` → `main`
   - Describe your changes
   - Request review from team

7. **Code Review**:
   - Team reviews, requests changes, or approves
   - Once approved, **merge the PR to main** on GitHub

8. **In Fabric, sync the shared Dev workspace**:
   - Open the shared `MyProject-Dev` workspace
   - Click **Source control** → **Updates** → **Update all**
   - This pulls your merged changes from `main`

### Alternative: Branch-Out to Fabric Workspace

If you prefer to develop partially or fully in Fabric UI:

1. **In Fabric, go to Source control → Branches**
2. Click **Branch out to another workspace**
3. Creates a new workspace + Git branch for your feature
4. Work there, commit to your feature branch
5. Create PR on GitHub
6. Once merged, the shared Dev workspace can update from `main`
7. Delete the temporary feature workspace

This hybrid approach is useful for teams with mixed developers (some UI-based, some code-based).

---

## End-to-End Example: Multiple Developers → Test Promotion

**Scenario**: Alice and Bob both work on the same project. How does their work flow to Test?

### Day 1: Two developers, two features

```
Main Branch (main)
├── existing analytics code

Alice creates feature branch:
├── feature/sales-dashboard
│   ├── workspace: MyProject-Dev-feature-alice
│   └── commits: "Add dashboard", "Fix styling"

Bob creates feature branch:  
├── feature/inventory-model
    ├── workspace: MyProject-Dev-feature-bob
    └── commits: "Add fact table", "Add relationships"
```

### Day 2: Alice finishes her dashboard

```
Alice's feature branch:
├── feature/sales-dashboard
    ├── Create PR on GitHub: feature/sales-dashboard → main
    ├── Bob reviews and approves
    └── Alice clicks "Merge PR"
    
Main Branch now contains:
├── Alice's 2 commits (dashboard + styling)
└── Bob's work still in feature/inventory-model branch (not merged yet)

Shared Dev Workspace:
├── Connected to: main branch
└── **NEEDS UPDATE**: Click Source control → Updates → Update all
```

After Dev workspace updates:
```
Shared Dev Workspace (MyProject-Dev) now contains:
├── Alice's dashboard ✅
├── Bob's inventory model ❌ (still waiting on his PR merge)
```

### Day 3: Bob finishes his model, both work ready for Test

```
Bob creates PR: feature/inventory-model → main
Alice reviews and approves
Bob merges PR

Main branch now has:
├── Alice's dashboard
└── Bob's inventory model

Shared Dev Workspace:
├── Update all from main
└── Now contains BOTH features ✅

Deployment Pipeline promotion:
├── Open pipeline
├── Dev stage: Shows both features
├── Click Deploy to Test
└── Test workspace now has both features ✅
```

### Key Insight

- **Individual commit selection DOESN'T HAPPEN at promotion time**
- It happens earlier: at the **GitHub PR approval step**
- Once commits merge to `main`, they're automatically included in deployments
- If you don't want something deployed, don't merge that PR to `main` (yet)

---

## When and Where to Use fabric-cicd Library

The **fabric-cicd** Python library is used for **Option 2** (Git-based build environments) and **Option 4** (ISV multi-tenant deployments). It's NOT needed for Developer #1 or #2 workflows above—they use Fabric's built-in Git integration and deployment pipelines.

### Use fabric-cicd When:

1. **You need a build environment** to transform item definitions before deployment (e.g., parameterize connectionIds, lakehouseIds, or database endpoints)
2. **You're following trunk-based Git strategy** (single main branch, all deployments from main)
3. **You want automated, repeatable deployments** via Python scripts in CI/CD pipelines (Azure DevOps, GitHub Actions)

### Where fabric-cicd Fits:

```
GitHub (main branch)
    ↓
[Build Pipeline - runs fabric-cicd script]
    ├─ Fetches item definitions from Git
    ├─ Runs parameterization (e.g., swap DEV connectionId → TEST connectionId)
    ├─ Publishes to Test workspace via fabric-cicd
    ↓
[Release Pipeline - runs fabric-cicd script]
    ├─ Copies items from Test to Prod
    ├─ Updates Prod-specific parameters
    ├─ Publishes via fabric-cicd
    ↓
Production Workspace
```

### Basic fabric-cicd Example:

```python
from fabric_cicd import FabricWorkspace, publish_all_items, unpublish_all_orphan_items

# Deploy to Test workspace
target_workspace = FabricWorkspace(
    workspace_id = "test-workspace-id",
    repository_directory = "/path/to/git/repo/fabric-items",
    item_type_in_scope = ["Notebook", "DataPipeline", "Environment", "SemanticModel"],
)

publish_all_items(target_workspace)                  # creates/updates all items
unpublish_all_orphan_items(target_workspace)        # deletes items no longer in Git
```

---

## Branching Best Practices

### For Developer #1 (Workspace-based):

**Strategy: Single main branch + feature branches**

1. **Main branch** = Dev workspace connected branch
2. **Feature branches** = Temporary branches for work-in-progress
3. **Workflow**:
   ```
   main (Dev workspace)
     ├─ feature/add-fact-table (temporary workspace)
   ```
   - Branch out to new workspace before starting feature work
   - Commit to feature branch
   - Create PR to main in GitHub
   - Review and merge PR
   - Delete feature branch
   - Update main workspace

### For Developer #2 (VS Code-based):

**Strategy: Gitflow or Trunk-based**

**Option A: Gitflow** (if using multiple primary branches)
```
main (Production-ready)
  ├─ develop (next release)
      ├─ feature/new-dashboard
      ├─ feature/model-update
release/v1.0
hotfix/bug-fix
```

**Option B: Trunk-based** (simpler, recommended)
```
main (always shippable)
  ├─ feature/user-initials
  ├─ feature/another-change
```

### Branching Limits:
- Branch names: max 244 characters
- Keep item count under 1,000 per workspace/branch
- Commit size limits: 50–125 MB (depending on provider)

---

## Manual Promotion: Dev → Test → Prod

### Using Deployment Pipelines (Recommended)

Before promoting, confirm this checklist:
1. All intended PRs are merged to `main`
2. Shared Dev integration workspace has been updated from Git (`Update all`)
3. Shared Dev workspace is the one assigned to Development stage

**Dev → Test:**
1. Open deployment pipeline in Fabric
2. In **Development** stage, click **Deploy** button
3. Select **Full deployment** (all items) or **Selective deployment** (pick specific items)
4. Review scope
5. Click **Deploy**
6. Wait for completion (usually 1–5 minutes)

**Test → Prod:**
1. In **Test** stage, click **Deploy**
2. (Optional) Set **deployment rules** to override environment-specific settings:
   - Example: Change `database_conn_id` from test database to prod database
   - Rules are applied automatically during deployment
3. Click **Deploy**

**Key Benefits:**
- Deployment history visible in Fabric UI
- Paired items automatically tracked (no duplicates)
- Rollback to previous deployments available
- Can deploy backward (later stage → earlier stage) if needed

### Without Deployment Pipelines (Manual Git approach):

**Dev → Test via Git:**
1. Create release branch from main: `release/to-test`
2. Cherry-pick commits to release branch (contains only items for promotion)
3. Create PR to a separate `test` branch
4. Merge PR
5. In Test workspace: **Update all** from `test` branch
6. Delete release branch

**Not recommended** because:
- No deployment history in Fabric UI
- Manual tracking of what was deployed
- Higher risk of configuration drift

---

## Configuration Management (Deployment Rules)

When promoting across environments, you often need different settings. **Deployment rules** automate this:

| Config | Dev | Test | Prod |
|--------|-----|------|------|
| Database | `dev-db-conn-id` | `test-db-conn-id` | `prod-db-conn-id` |
| Lakehouse | `dev-lh.OneLake` | `test-lh.OneLake` | `prod-lh.OneLake` |
| Gateway | dev-gateway | test-gateway | prod-gateway |

**Set rules in Deployment Pipeline**:
1. Go to **Production** stage → settings
2. Define rule: `"Semantic Model X" → database parameter = "prod-db-conn-id"`
3. During Test → Prod deployment, rule auto-applied
4. No manual updates needed

---

## Quick Decision Matrix

| Scenario | Developer #1 | Developer #2 | Tool |
|----------|--------------|--------------|------|
| Work in browser UI | ✅ | ❌ | Git Integration + Deployment Pipelines |
| Work in VS Code | ❌ | ✅ | Git Integration + Deployment Pipelines |
| Multiple branches/PRs | ✅ (branch-out) | ✅ (native Git) | Git Integration |
| Parameterized deployments | ❌ | ✅ (with scripts) | fabric-cicd (optional) |
| Simple Dev/Test/Prod | ✅ | ✅ | Deployment Pipelines |
| ISV multi-tenant SaaS | ❌ | ✅ | fabric-cicd + Option 4 |

---

## Summary

### The Core Architecture

The secret to understanding Fabric deployments is this separation:

1. **Git Integration** = **developer collaboration** (branches, PRs, merges to main)
2. **Deployment Pipelines** = **environment promotion** (Dev workspace → Test workspace → Prod workspace)

They work together but serve different purposes:
- Git handles "*should this code/definition be included in the next release?*" (answered via PR reviews)
- Deployment pipelines handle "*copy everything from stage A to stage B*" (answered via manual button click or API)

### Multi-Developer Workflow Summary

```
Developer #1              Developer #2
  ↓ feature branch          ↓ feature branch
   ↓ (optional personal WS)  ↓ (optional personal WS)
  ↓ commit + PR            ↓ commit + PR
  ↓ (reviewed)             ↓ (reviewed)
  ↓ MERGE to main          ↓ MERGE to main
  ↓                        ↓
  ←── Shared Dev Workspace syncs from main ──→
              ↓
  ←── Deployment Pipeline promotes to Test ──→
              ↓
        Test Workspace
```

### What This Means for You

- **Both developers** use feature branches (isolated work)
- **Each developer can optionally use their own workspace** for isolation
- **Team controls promotion via Git PRs** (code review gate)
- **Shared Dev workspace is the source of truth** for deployments (always on `main`)
- **Deployment pipelines handle the mechanics** (copy everything Dev → Test → Prod)
- **fabric-cicd is optional** (only needed if you want custom parameterization or selective item deployment)

For typical teams: Use **Option 3** (Git Integration + Deployment Pipelines). Simple, built-in, no fabric-cicd needed.

---

## FAQ: Selecting Specific Commits for Promotion

**Question**: "What if I only want to promote certain developer's work to Test, not everything?"

**Answer**: Deployment pipelines deploy **entire workspaces**, not individual commits. Here are your options:

### Option 1: Gate at the Git PR Level (Recommended for most teams)

Control which commits go to `main` through PR approvals:
- Development work on feature branches stays separate
- Only approved PRs → merge to `main`
- Only `main` changes → get deployed
- If some changes aren't ready, don't merge that PR yet

```
feature/dashboard (ready)      → Merge PR → main → Deploy to Test ✅
feature/reports (not ready)    → Hold PR (don't merge) ↓ stays off main
feature/security-fix (urgent)  → Merge PR → main → Deploy to Test ✅

Result: Only 'dashboard' and 'security-fix' reach Test, 'reports' waits
```

### Option 2: Use Separate Release Branches (For staged promotions)

If you want multiple features at different readiness levels:

1. Create a `release/v1.0` branch from `main`
2. Cherry-pick only the commits/PRs you want to promote
3. Create a secondary Dev workspace connected to `release/v1.0`
4. Use that workspace for Test promotion
5. When ready, merge `release/v1.0` back to `main`

### Option 3: Use fabric-cicd for Selective Deployments

If you need programmatic control over which **items** (not commits) deploy:

```python
from fabric_cicd import FabricWorkspace, publish_all_items

# Deploy only Notebooks and Semantic Models, skip Pipelines
target_workspace = FabricWorkspace(
    workspace_id = "test-workspace-id",
    repository_directory = "/path/to/repo",
    item_type_in_scope = ["Notebook", "SemanticModel"],  # exclude DataPipeline
)
publish_all_items(target_workspace)
```

This lets you deploy by **item type**, but not by individual commits.

### Option 4: Use Bulk Import API with Custom Scripts

For maximum control over which specific items deploy:
- Write a custom script that reads item definitions from Git
- Filter by item name, type, or custom metadata
- Use Bulk Import API to deploy only filtered items

⚠️ **This is complex** and recommended only for ISVs or scenarios with many items per commit.
