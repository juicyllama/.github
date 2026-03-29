# Git Workflow & Commit Convention

## 1. Branch Strategy

| Concept | Rule |
| :--- | :--- |
| **Source of truth** | `main` — always deployable, production-ready |
| **Staging** | `staging` — for deeper testing only (backend, product review).<br>Kept in sync with `main` via [GitHub Action](https://github.com/juicyllama/helpful-github-actions/blob/main/.github/workflows/sync-main-to-stage.yml) |
| **Feature branches** | Created from `main`, merged back to `main` |
| **Hotfixes** | Created from `main`, merged to `main`. Staging auto-syncs |

### Branch Naming

| Type | Format | Example |
| :--- | :--- | :--- |
| Feature | `feature/XXX-NNN` | `feature/JL-369` |
| Bug fix | `bugfix/XXX-NNN` | `bugfix/JL-421` |
| Hotfix | `hotfix/XXX-NNN` | `hotfix/JL-500` |

---

## 2. Workflow Diagram

```text
main (source of truth)
 |
 |--- feature/XXX
 |      |
 |      | (develop, commit, push)
 |      |
 |<-- PR --| (merge to main)
 |
 |--> staging (auto-synced after prod deploy)
 |      |
 |      | (deeper testing if needed)
 |      |
 |      NEVER merge staging → main
 |
```

---

## 3. Step-by-Step Flow

### Regular Development (Feature / Bugfix)

1. **Create branch from `main`**
   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/XXX
   ```
2. **Develop & commit** (follow commit convention below)
3. **Push & create PR to `main`**
   ```bash
   git push origin feature/XXX
   ```
   Create PR: `feature/XXX` → `main`
4. **Need staging testing?** Create a *separate* PR from the same branch to `staging`, or cherry-pick. Do **NOT** merge staging into your feature branch.
5. **After PR approval & merge** — delete the remote branch
   ```bash
   git push origin --delete feature/XXX
   ```
   *Or enable “Automatically delete head branches” in GitHub repo settings.*

### Hotfix (Urgent Production Fix)

1. **Create branch from `main`**
   ```bash
   git checkout main
   git pull origin main
   git checkout -b hotfix/XXX
   ```
2. **Fix, commit, push**
3. **PR to `main`** — get quick review, merge
4. **Staging auto-syncs** after prod deploy succeeds
5. **Delete remote branch** after merge

---

## 4. Golden Rules

| # | Rule |
| :--- | :--- |
| 1 | **Everything starts from `main`** |
| 2 | **NEVER merge `staging` → `main`** |
| 3 | **NEVER merge `staging` into your feature branch** (causes commit pollution) |
| 4 | **Delete remote branches after merge** to keep the repo clean |
| 5 | **PR to `staging` only when needed** for deeper testing (backend, product sign-off) |
| 6 | **Staging auto-syncs** with main after every prod deploy. If conflict, use manual sync (see Section 9) |

---

## 5. Commit Message Convention

### Format

```text
type(scope): short description

- what was done
- what was done

Refs: XXX
```

### Types

| Type | When to use |
| :--- | :--- |
| `feat` | New feature or functionality |
| `fix` | Bug fix |
| `refactor` | Code restructuring (no behavior change) |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `docs` | Documentation changes |
| `chore` | Build, CI, tooling, dependencies |
| `style` | Formatting, whitespace (no logic change) |
| `build` | Build system changes |
| `ci` | CI/CD configuration |
| `revert` | Revert a previous commit |
| `add` | Add something new (non-feature) |
| `update` | Update existing functionality |

### Scope (required)

Use the feature area: `chat`, `images`, `video`, `auth`, `admin`, `prompts`, `dashboard`, `billing`, `teams`, `db`, `api`, `ci`, `ui`, `ai`, `personas`, `artifacts`, `credits`, `webhook`, `email`, etc.

### Refs footer

**Accepted formats** (case-insensitive, space after colon optional):
* `Refs: XXX-421`
* `refs: XXX-421`
* `Refs: XXX-421`

### Recommendations

* **Subject line:** lowercase, no period at end, max 72 chars
* **Scope:** **required**, lowercase
* **Body:** bullet points with `-` prefix describing what was done
* One logical change per commit

### Enforcement

* **Local:** Husky pre-commit hook validates format on every commit
* **Server:** `pr-lint.yml` GitHub Action validates PR title + Refs on all PRs to `main`
* Both must pass — cannot be bypassed without `--no-verify` (local) or admin override (server)

### Examples

```text
feat(dashboard): reorder mobile layout - actions above stats

- move QuickActions above StatsCards on mobile
- keep desktop layout unchanged
- fix QuickActionCard height consistency

Refs: XXX-369
```

```text
fix(prompts): change category filter to include logic

- remove auto-selection of all categories on load
- empty selection shows all prompts
- fix infinite re-render loop in subcategory effect

Refs: XXX-370
```

```text
chore(ci): update node version to 22
```
*(No Refs required for chore type)*

---

## 6. PR Guidelines

* **Title:** Same format as commit message — `type(scope): description`
* **Description:** What changed, why, how to test
* **Refs:** Include `Refs: XXX` in PR body (enforced by `pr-lint.yml` for feature/fix types)
* **Reviewers:** AI Review and ask others to review if required
* **Target branch:** Always `main` (unless staging-only testing)

---

## 7. Pre-Commit Checklist

Before committing, run to catch issues early:

```bash
npm run go 
```
---

## 8. CI/CD Pipeline

### How it works

| Event | What happens | Deploy? |
| :--- | :--- | :--- |
| PR to `main` | Lint → Type Check → Unit Tests → Build | No |
| Push to `main` | Lint → Type Check → Unit Tests → Build → Deploy (Prod) → Auto-sync staging | Yes (prod) |
| Push to `staging` | Lint → Type Check → Unit Tests → Build → Deploy (Preview) | Yes (preview) |

**If any step fails, the deploy does NOT happen.** The code will not reach production or staging until all checks pass.


### Where to monitor

* **Build & deploy status:** [GitHub Actions](#)
* **Railway deployments:** [Railway Dashboard](#)

### Railway auto-deploy

Railway auto-deploy is **enavled** for `main` and `staging` branches. Deploys happen automatically once CI passes.

---

## 9. Staging Sync

### Automatic (after every prod deploy)

The `sync-staging` job in `release.yml` automatically merges `main` into `staging` after every successful production deploy. This preserves any staging-only commits (e.g., features being tested by PM).

If there is a merge conflict, auto-sync **skips silently** — no force push. Use manual sync below.

### Manual (GitHub Actions UI)

Go to **Actions → Sync Staging → Run workflow** and choose a strategy:

| Strategy | What it does | When to use |
| :--- | :--- | :--- |
| `merge` | Merge main into staging (preserves staging commits) | Default. Safe option |
| `force` | Reset staging = main (overwrites everything) | When staging is messy or diverged too far |
