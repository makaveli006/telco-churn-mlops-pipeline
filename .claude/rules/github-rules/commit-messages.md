# Git Commit Message Best Practices

## Standard Format

```
<type>(<scope>): <short, imperative summary>
```

## Commit Types

| Type | Purpose | Example |
|------|---------|---------|
| `feat` | New feature | `feat(jobs): add Ashby ATS fetcher` |
| `fix` | Bug fix | `fix(pipeline): handle empty slug list gracefully` |
| `hotfix` | Urgent production fix | `hotfix(api): resolve null pointer in auth middleware` |
| `refactor` | Code restructure, no behavior change | `refactor(normalizer): simplify salary extraction logic` |
| `perf` | Performance improvement | `perf(search): cache DuckDuckGo results per run` |
| `docs` | Documentation only | `docs(readme): add backend setup steps` |
| `style` | Formatting/UI, no logic change | `style(ui): fix spacing in job card component` |
| `test` | Add or modify tests | `test(ingestion): add Ashby fetcher unit tests` |
| `chore` | Maintenance, deps, CI | `chore(deps): add duckduckgo-search>=8.0.0` |
| `release` | Version/deployment commit | `release: production deployment v1.0.12` |

## Examples

```bash
# Feature
feat(ingestion): add Ashby ATS job fetcher

# Bug fix
fix(pipeline): resolve KeyError when greenhouse token is empty

# Hotfix
hotfix(api): handle null session in auth middleware

# Refactor
refactor(config): extract slugs_list into reusable property

# Chore
chore(ci): update GitHub Actions to Node 20
```

## Rules

**DO:**
- Use imperative tone: `"Add validation"` not `"Added validation"`
- Keep summary under 50 characters
- Capitalize the first letter
- One logical change per commit

**DON'T:**
- Write vague messages like `"update code"` or `"fix issues"`
- Include multiple unrelated changes in one commit
- Push `WIP` commits to shared branches (`develop`, `main`)
