# SkillAra — Repository map

Quick reference for which local folder maps to which GitHub repository.

| Local folder | GitHub repository | Purpose |
|--------------|-------------------|---------|
| *(this root, docs only)* | [Varshini1812/SkillAra](https://github.com/Varshini1812/SkillAra) | Platform overview & resume hub |
| `SkillAra_server/` | [Varshini1812/SkillAra_server](https://github.com/Varshini1812/SkillAra_server) | Backend API |
| `SkillAra_adminpanel/` | [Varshini1812/SkillAra_adminpanel](https://github.com/Varshini1812/SkillAra_adminpanel) | Platform super-admin UI |
| `SkillAra_client/` | [Varshini1812/SkillAra_tenantUI](https://github.com/Varshini1812/SkillAra_tenantUI) | Tenant UI (students + org admin) |

## Initializing the umbrella repo (GitHub: SkillAra)

This repo should contain **only** documentation — no application source code.

```bash
cd SkillAra
git init
git add README.md docs/ REPOSITORIES.md
git commit -m "Add SkillAra platform overview and repository links"
git branch -M main
git remote add origin https://github.com/Varshini1812/SkillAra.git
git push -u origin main
```

## Pushing tenant UI (client) changes

```bash
cd SkillAra_client
git init   # only if not already a repo
git remote add origin https://github.com/Varshini1812/SkillAra_tenantUI.git
git add .
git commit -m "Describe your changes"
git push -u origin main
```

If `git remote -v` shows a different URL, either change the remote or push to the URL you intend:

```bash
git remote set-url origin https://github.com/Varshini1812/SkillAra_tenantUI.git
```

## Resume one-liner

> **SkillAra** — Multi-tenant SaaS LMS with subdomain isolation, embedded RBAC, platform/tenant admin split, and OpenAI-powered learning tools.  
> Hub: [github.com/Varshini1812/SkillAra](https://github.com/Varshini1812/SkillAra)
