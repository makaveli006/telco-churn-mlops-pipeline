# Git Branching Model (Gitflow)

## Main Branches

| Branch | Purpose | Deployed To |
|--------|---------|-------------|
| `main` | Stable production code — only updated via release or hotfix merges | Production |
| `develop` | Integration and QA — all feature branches merge here | Test/Staging |

## Supporting Branch Types

| Branch | Created From | Merged Into | Naming |
|--------|-------------|-------------|--------|
| Feature | `develop` | `develop` | `feature/1234-short-description` |
| Release | `develop` | `main` + `develop` | `release/1.0.x` |
| Hotfix | `main` | `main` + `develop` | `hotfix/1.0.x` |

## Feature Branch Flow

```bash
git checkout develop
git checkout -b feature/2501-fix-quotation-error

# ... do the work ...

git checkout develop
git merge feature/2501-fix-quotation-error
# → auto-deploy to test
```

## Release Branch Flow

```bash
git checkout develop
git checkout -b release/1.0.11
# → deploy to UAT, run final QA

git checkout main
git merge release/1.0.11
git tag -a v1.0.11 -m "Production release 1.0.11"
git push origin main --tags
# → production deploy

git checkout develop
git merge release/1.0.11
```

## Hotfix Branch Flow

```bash
git checkout main
git checkout -b hotfix/1.0.12

# ... fix the issue ...

git checkout main
git merge hotfix/1.0.12
git checkout develop
git merge hotfix/1.0.12
```

## Best Practices

- **Always tag production releases** (e.g. `v1.0.11`) for easy rollback
- **Protect branches**: require PR before merging into `develop` or `main`; only release managers merge into `main`
- **Automate deployments**:
  - Push to `develop` → deploy to test
  - Merge into `main` → deploy to production
- **Keep release notes tied to git tags** for traceability
