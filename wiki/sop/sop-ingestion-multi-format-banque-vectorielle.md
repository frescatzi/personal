---
type: wiki
title: SOP — Ingestion multi-format (texte/markdown + PDF/dossier) vers banque vectorielle
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--pos-generique-ingestion-multi-format-vers-banque-vectorielle.md
  - raw/2026-07-06--pos-generique-ingestion-multi-format-vers-banque-vectorielle.md
related:
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/concept-memoire-vivante-agents.md
  - wiki/sop/sop-reparer-webhook-n8n-ingestion-pdf.md
  - wiki/sop/sop-diagnostiquer-pipeline-memoire-vectorielle.md
  - wiki/sop/SOP_installer-pgvector-sur-postgres-coolify.md
updated: 2026-07-06
---

# SOP — Ingestion multi-format (texte/markdown + PDF/dossier) vers banque vectorielle

## Principe

Patron réutilisable pour alimenter une banque vectorielle (pgvector) depuis (A) du texte/markdown fourni directement ou (B) un dossier de fichiers avec récursion.

## Cœur commun (les deux entrées partagent ce pipeline)

```
Chunk → Embed → Build Insert (paramétré) → Postgres Insert
```

- **Chunk** : découpe le texte (`size` 3000 / `overlap` 200 recommandés), propage les métadonnées (`brand`, `title`, `source_ref`, `collection`).
- **Embed** : 1 appel au modèle d'embedding **par chunk** — même modèle partout (écriture ↔ recherche). Distances incohérentes si modèles différents.
- **Build Insert** : aligne les embeddings avec les chunks par index (`$input.all()` ↔ `$('Chunk').all()`), résout la table via le registre (`memory_registry`), INSERT **paramétré** (`$1..$n`, hash de contenu, `ON CONFLICT content_hash DO NOTHING`).

## Entrée A — texte/markdown fourni

```
Trigger {brand, title, content, collection, source_ref}
→ Cœur
```

Appelable comme sous-workflow ou outil (ingérer une note, un doc converti, une source Obsidian).

## Entrée B — dossier de fichiers avec récursion

```
Trigger {folderId}
→ Google Drive Search (lister le dossier)
→ Filter (PDF + markdown/texte — exclure les non supportés)
→ Download (By ID = {{ $json.id }}, sortie binaire = "data")
→ Switch par mimeType
    ├─ branche PDF     → Extract From PDF (Input Binary Field = "data")
    └─ branche texte   → Extract from text file (Input Binary Field = "data")
→ normaliser en content (champ $json.text)
→ Cœur
```

Un node Extract ne supporte **qu'un seul type** — obligatoire de brancher.

## Garde-fous & pièges

| Risque | Fix |
|--------|-----|
| Format non supporté (.docx) | Convertir en amont (md/pdf) ou exclure dans le Filter |
| Perte de métadonnées après HTTP/Extract (le node remplace le json) | Ré-aligner via référence indexée au node Chunk/Filter (`$('Chunk').all()`) |
| TRUNCATE qui vide d'autres collections | Préférer l'idempotence via `ON CONFLICT content_hash DO NOTHING` |
| Requête `in parents` ne descend pas dans les sous-dossiers | Boucler par sous-dossier pour une récursion profonde |
| SQL concaténé (casse sur contenu riche) | Requêtes paramétrées (`$1..$n`) — voir [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] |
| Embedding unique non respecté | Forcer le même modèle partout dans le projet |

## Valider

1. Ingérer un fichier connu.
2. Vérifier le `count` par `source_ref`.
3. Rechercher un fait du document via le point de récupération (MCP/webhook) → doit remonter avec un bon score.

## Voir aussi

- [[concept-memoire-vectorielle-multi-marques]] — schéma banque, registre, multi-marques.
- [[concept-memoire-vivante-agents]] — primitives WRITE/READ/CONSOLIDATE + idempotence.
- [[sop/sop-reparer-webhook-n8n-ingestion-pdf]] — réparer un webhook last-node + ingestion fichier unique.
- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — diagnostic + SQL paramétré.
- [[sop/SOP_installer-pgvector-sur-postgres-coolify]] — setup de la base vectorielle.
