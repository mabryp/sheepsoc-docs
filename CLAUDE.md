# Sheepsoc Docs — Claude Context

## This Repo

This is the MkDocs documentation site for the sheepsoc homelab (`mabryp/sheepsoc-docs`).

| Key | Value |
|---|---|
| Docs source | `docs/` |
| Build config | `mkdocs.yml` |
| CI | Push to `main` → self-hosted GitHub Actions runner on sheepsoc → `mkdocs build` |
| Served at | `http://192.168.50.100/docs/` (LAN only) |

## Lab Facts

**All factual information about the sheepsoc lab lives in `docs/`.** Do not rely on
training data for current lab state — read the relevant page. The page registry in
`schema.md` tells you exactly where to find any topic.

## Wiki Maintenance Rules

Read `schema.md` before making any changes to `docs/`. It defines:

- **Page registry** — what every page covers
- **Relationship vocabulary** — how to describe links between pages
- **Link rules** — what links are required, what syntax to use, when to add backlinks
- **Update triggers** — what infrastructure changes require which pages to be updated
- **Page anatomy** — standard structure for service pages and runbooks
- **Conventions** — code blocks, admonitions, secrets, dates

## Behavior Rules

1. **Explain before acting** — describe what you are going to do and why before
   making changes
2. **Commit and push** — all changes must be committed and pushed to `origin/main`
   to trigger CI; do not run `mkdocs build` or `mkdocs serve` manually
3. **No secrets** — never write passwords, tokens, or API keys into any page; use
   `<password>` or `<token>` as placeholders
4. **Check update triggers** — when updating any page, read `schema.md` section 6
   to see if other pages also need updating
5. **Maintain interlinks** — when adding or editing a page, follow the link rules
   in `schema.md` section 4; always ask whether a backlink is needed
6. **Nothing happens without Phillip knowing** — all changes should be reviewed
   before being pushed to production unless operating under a standing permission grant

## Who You Are Working With

- User: Phillip Mabry (pmabry)
- Comfortable with Linux and Python; newer to conda — explain conda concepts clearly
- Preferred editor: vim
- Philosophy: nothing happens on this machine without Phillip knowing about it
