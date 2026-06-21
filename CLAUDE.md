# CLAUDE.md — Coffre `personal`

> **Schéma de CE coffre.** Zone *perso, business générique & connaissance générale*.
> Routage inter-coffres : `lumina-meta/ROUTING.md`. Coffre autonome (repo séparé). Source = ces fichiers ; GitHub = backup.

---

## 1. Périmètre

Tout savoir à conserver qui n'est ni une technique IA réutilisable, ni propre à une marque :
finances perso, santé, préceptes/notes de livres, business générique, productivité, notes de vie, références transverses.

**Hors périmètre →** technique IA → `ai-automation`. Spécifique marque → `brands`.

---

## 2. Structure interne

```text
personal/
├── CLAUDE.md     ← ce fichier
├── index.md      ← carte de navigation du wiki
├── log.md        ← journal du coffre
├── raw/          ← sources curées & immuables
└── wiki/         ← pages propres (LLM)
```

Optionnel : regrouper le `wiki/` par thèmes via des sous-dossiers (`finances/`, `sante/`, `livres/`) si le volume le justifie. À introduire seulement quand utile (pas au départ).

---

## 3. Conventions

Frontmatter `wiki/` :
```yaml
---
type: wiki
title: "Nom du concept / synthèse"
status: draft          # draft | active | retired
publish: none          # none | notion
vault: personal
brand: null
sources: [raw/2026-06-20--note.md]
updated: 2026-06-20
---
```

- `status` / `publish` : mêmes règles que les autres coffres.
- ⚠️ Ce coffre contient du **personnel/sensible** : par défaut `publish: none`. Ne publier vers Notion que ce que tu veux vraiment exposer.

---

## 4. Workflows

Identiques au modèle : voir `../ai-automation/CLAUDE.md` §4–6 pour le détail **Ingest / Q&A / Lint**.
Nuance confidentialité : au lint, signaler toute page sensible passée par erreur en `publish: notion`.

---

## 5. `log.md` — format
```text
## [2026-06-20] ingest | Préceptes de "Deep Work"
## [2026-06-20] qa     | "Règle d'épargne 50/30/20" → maj wiki/finances-budget
```

---

*CLAUDE.md `personal` v2.2 — 2026-06-20.*
