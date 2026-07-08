---
type: wiki
title: Concept — Validation du runner auto-ingest (raw → wiki → Git)
status: draft
publish: none
vault: ai-automation
brand: null
sources:
  - raw/raw/2026-07-06--test-auto-ingest.md
  - raw/2026-07-06--test-auto-ingest-2.md
  - raw/2026-07-06--test-3.md
related:
  - concept-pipeline-memoire-wiki-git
  - sop/sop-lumina-auto-ingest-raw-vers-wiki
  - sop/sop-generique-runner-llm-headless-webhook
  - concept-archivage-n8n-idempotent
updated: 2026-07-07
---

# Concept — Validation du runner auto-ingest (raw → wiki → Git)

> Page reconstituée le 2026-07-07 : le contenu original avait été rédigé en plusieurs passes (voir `log.md`, entrées 2026-07-06) mais le fichier avait disparu du disque sans que la page ne soit recréée — backlink cassé corrigé ici. Les fichiers `raw/` sources ont depuis été archivés (leur contenu de test n'est plus disponible verbatim), la reconstitution s'appuie sur l'historique de `log.md`.

## Objet

Artefact de test qui a servi à valider **de bout en bout** le pipeline d'auto-ingest Lumina : dépôt d'un fichier de test dans `raw/`, déclenchement automatique (push → webhook n8n → runner), compilation `raw/ → wiki/` par Claude, mise à jour de `index.md`/`log.md`/`_archive_queue.json`, puis archivage.

Voir l'implémentation validée : [[sop/sop-lumina-auto-ingest-raw-vers-wiki]] (Lumina) et [[sop/sop-generique-runner-llm-headless-webhook]] (patron générique).

## Ce qui a été validé (runs 2026-07-06)

1. **Déclenchement manuel** (`/ingest`) sur un premier fichier de test → page wiki créée en `draft`, `index.md` et `log.md` mis à jour.
2. **Déclenchement automatique** (push GitHub → webhook n8n → runner) sur une reformulation du même test → **dédup** correcte : pas de nouvelle page, mise à jour de `sources:` et du corps de cette page à la place.
3. **Re-capture à la racine `raw/`** d'un contenu déjà compilé depuis `raw/raw/…` → dédup par empreinte confirmée (corps identique, chemin différent).
4. **Purge du manifeste `_archive_queue.json`** confirmée : entrées absentes du disque (déjà archivées par n8n) retirées après vérification de sync (`HEAD == origin/main`), jamais sur une simple hypothèse.

## Conclusion

Le pipeline auto-ingest (dédup par empreinte, anti-boucle, purge du manifeste basée sur l'état disque réel) fonctionne comme conçu. Cette page reste `draft` : elle documente un artefact de test, pas un concept à promouvoir en l'état.

## Voir aussi

- [[concept-pipeline-memoire-wiki-git]] — le pipeline global dont ce test valide le maillon `raw/ → wiki/`.
- [[concept-archivage-n8n-idempotent]] — le garde-fou de purge du manifeste testé au point 4.
