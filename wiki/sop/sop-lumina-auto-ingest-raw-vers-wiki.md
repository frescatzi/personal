---
type: wiki
title: "SOP Lumina — Auto-Ingest raw→wiki (runner Claude headless, multi-coffres)"
status: draft
publish: none
vault: ai-automation
brand: null
sources: ["raw/2026-07-07--pos-lumina-auto-ingest-raw-vers-wiki-2026-07-07.md"]
related: ["sop/sop-generique-runner-llm-headless-webhook", "concept-pipeline-memoire-wiki-git", "sop/sop-lumina-archive-raw-vers-drive", "concept-archivage-n8n-idempotent", "concept-validation-auto-ingest"]
updated: 2026-07-07
---

# SOP Lumina — Auto-Ingest raw→wiki (runner Claude headless, multi-coffres)

> **Statut :** construit, testé de bout en bout, **opérationnel sur les 3 coffres** (`ai-automation`, `brands`, `personal`) — v2, 2026-07-07. Remplace une v1 mono-coffre du 2026-07-06.
> Implémentation Lumina concrète du patron générique [[sop/sop-generique-runner-llm-headless-webhook]], appliquée à l'étape 3 (`raw/ → wiki/`) de [[concept-pipeline-memoire-wiki-git]] — **désormais automatisée**, alors qu'elle était manuelle.

---

## 1. Place dans le pipeline

```
Drive Inbox → intake n8n → GitHub raw/ ──push──▶ n8n Webhook (Auto-Ingest)
                                                    │ filtre main + regex raw/**.md → POST /ingest {repo, files}
                                                    ▼
                                    Runner Claude headless (Coolify)
                                    git reset --hard origin/main → claude -p (sonnet, ciblé) → maj index.md/log.md/_archive_queue.json → commit+push
                                                    │
                        push wiki/** ───────────────┴──▶ pgvector + Notion
                        cron d'archivage ────────────────▶ vide raw/ vers Drive
```

C'est le chaînon automatisé manquant entre l'intake (remplit `raw/`) et l'archivage ([[sop/sop-lumina-archive-raw-vers-drive]], qui vide `raw/`).

## 2. Architecture

- **Webhook GitHub** sur chacun des 3 repos (`frescatzi/ai-automation`, `brands`, `personal`) → workflow n8n `LUMINA — Auto-Ingest — Raw→Wiki` (Active), URL `n8n.aftersunpeople.com/webhook/lumina-auto-ingest`.
- **n8n** : `Webhook (réponse immédiate) → Code (filtre anti-boucle) → HTTP Request /ingest`. Le filtre ne garde que la branche `main` et au moins un fichier ajouté/modifié dont le chemin matche `/(^|\/)raw\//` et finit par `.md` (regex profondeur-agnostique — capte aussi `<MARQUE>/raw/…` pour le coffre `brands`). Les suppressions (archivage) et les commits du bot sont ignorés.
- **Runner** : service HTTP (`GET /health`, `POST /ingest`) sur `runner.aftersunpeople.com` (Coolify/Hetzner), déployé en **Dockerfile sans dépôt Git dédié** (code inline : `server.js` + `package.json` + `ingest-prompt.md`). Étapes de `/ingest` : auth par header `X-Ingest-Secret` → verrou anti-concurrence → validation `repo` contre une allowlist (`ai-automation`, `brands`, `personal`) → sync dure (`git reset --hard origin/main`) → `claude -p "$(cat ingest-prompt.md + liste des fichiers changés)" --model sonnet --permission-mode bypassPermissions --allowedTools Read Write Edit Bash Grep Glob --max-turns 30` → si diff non vide : commit + `pull --rebase` + push → réponse `{status, repo, changed, files, summary}`.
- **Prompt d'ingest** : générique multi-coffres, s'appuie sur le `CLAUDE.md` propre à chaque coffre pour les règles locales (structure, marques, périmètre) ; impose les garde-fous universels (dédup, synthèse, frontmatter, jamais `active`/`notion`, `raw/` intact, exclusions passations/hors-périmètre, maj `index.md`/`log.md`, append `_archive_queue.json`). Le runner y injecte en fin de prompt la liste exacte des fichiers changés → Claude ne compile que ceux-là.

## 3. Secrets (gate CEO — jamais dans le chat/repo)

| Secret | Où | Rôle |
|---|---|---|
| `ANTHROPIC_API_KEY` | env Coolify | fait tourner Claude Code |
| `GIT_PAT` | env Coolify | token seul, **Contents: Read and write** sur les 3 repos |
| `INGEST_SECRET` | env Coolify + credential/header n8n | authentifie n8n → runner |

⚠️ Coolify passe les env en **build-args** → visibles en clair dans les logs de déploiement. Après tout changement, considérer les secrets exposés et les **faire tourner** (révoquer/régénérer + redeploy).

## 4. Garde-fous automatiques (remplacent la revue humaine)

1. Dédup par empreinte (même contenu, date différente → pas de nouvelle page).
2. Anti-boucle (déclenchement sur `raw/**.md` ajouté/modifié seulement ; commits du bot et suppressions d'archivage ignorés).
3. Verrou de concurrence (un seul ingest à la fois).
4. Plafond `--max-turns 30` + compilation ciblée aux fichiers listés.
5. `status: draft` / `publish: none` par défaut → promotion vers les agents/Notion **toujours manuelle**.
6. Runner = seul writer de `wiki/`, avec `git pull --rebase` avant push pour absorber un commit concurrent (Obsidian Git).
7. Allowlist des 3 repos autorisés.

## 5. Pièges rencontrés & solutions

| Piège | Solution |
|---|---|
| Claude Code refuse de tourner en root | Utilisateur `node` (UID 1000 de `node:20-slim`) + `chown -R node:node`. |
| Volume monté sur `/vault` (root-only) → écriture impossible | Pas de volume : `/vault` éphémère, re-cloné à chaque conteneur. |
| Placeholders au lieu des vraies valeurs de secrets | Coller les vraies valeurs ; vérifier le préfixe (`sk-ant-…`). |
| PAT sans `Contents: write` → 403 au push | Fine-grained token, repos ciblés, scope Read+write. |
| Oubli de « Save » avant « Deploy » (Coolify) | Toujours Save avant Deploy. |
| Changement d'env non pris en compte | Redeploy après tout changement de variable. |
| `brands` range les raw sous `<MARQUE>/raw/` (+ doublons de casse) | Filtre regex profondeur-agnostique ; consolider les doublons de casse. |
| Fichier créé sans `.md` ou dans `raw/raw/` | Créer depuis la racine ou taper le nom dans le bon dossier. |
| Clone partiel bloquant | Nettoyer `/vault` avant de re-cloner. |

## 6. Tests de recette (validés)

1. Push d'un `.md` dans `raw/` (ou `<marque>/raw/`) → n8n en ~2 s → runner répond `ok` → page wiki (draft) ou ligne log poussée (testé OK sur `brands`).
2. Idempotence : re-pousser le même contenu → aucune nouvelle page.
3. Anti-boucle : le commit du bot ne redéclenche pas d'ingest.
4. Sécurité : `INGEST_SECRET` faux → `403`.

## 7. Exploitation & sécurité

- Rotation des secrets après toute exposition (logs Coolify) : régénérer PAT + `INGEST_SECRET` (Coolify **et** n8n) → redeploy → valider par un push test.
- Promotion vers les agents : toujours manuelle (`draft → active` + `publish: notion`).
- Nettoyage : les fichiers de test dans `raw/` sont balayés par le cron d'archivage ([[sop/sop-lumina-archive-raw-vers-drive]]).

## 8. Évolutions prévues

- Promotion assistée (`draft → active` + `publish: notion` sur demande).
- Schedule de secours (horaire) en cas de webhook manqué.
- Revue optionnelle via branche `auto-ingest` + PR.
- Vérification HMAC de la signature du webhook GitHub dans n8n.

---

## Voir aussi

- [[sop/sop-generique-runner-llm-headless-webhook]] — le patron générique dont ce runner est l'implémentation concrète Lumina.
- [[concept-pipeline-memoire-wiki-git]] — l'étape 3 (`raw/ → wiki/`) que ce runner automatise désormais.
- [[sop/sop-lumina-archive-raw-vers-drive]] — le workflow n8n aval qui consomme `_archive_queue.json` produit par ce runner.
- [[concept-archivage-n8n-idempotent]] — garde-fous équivalents côté archivage.
- [[concept-validation-auto-ingest]] — validation de bout en bout des runs précédents de ce runner.
