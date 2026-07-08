---
type: wiki
title: Concept — Archivage idempotent post-compilation (pattern n8n)
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-06-29--howto-generique-archiver-source-apres-compilation-idempotent-n8n.md
  - raw/2026-06-29--sop-generique-workflow-archivage-idempotent-n8n.md
related:
  - sop/sop-lumina-archive-raw-vers-drive
  - concept-intake-source-git
  - synthese-lumina-systeme-reference
updated: 2026-06-29
---

# Concept — Archivage idempotent post-compilation (pattern n8n)

> **Principe directeur :** *l'amont étiquette (manifeste), n8n exécute.* Le process amont (agent LLM, Claude Code…) écrit un manifeste listant les fichiers prêts à archiver ; n8n fait le déplacement physique — sans perte ni doublon.

Voir [[sop/sop-lumina-archive-raw-vers-drive]] pour l'implémentation Lumina concrète.

---

## Contexte & objectif

Après qu'un agent a compilé des sources brutes (`raw/`) en livrables (`wiki/`), le brut doit être :
1. **archivé** dans un stockage de sécurité (Drive, S3…),
2. **supprimé de la source** pour garder le repo propre.

L'enjeu : ne jamais perdre un fichier, ne jamais créer de doublon, pouvoir re-lancer le workflow à tout moment sans effets de bord.

---

## Le pattern — 6 nodes

```
Trigger
  → Lire le manifeste (texte)
  → Décoder + parser → 1 item par fichier
  → Récupérer fichier source (binaire)  ──Success──→ Uploader vers archive ──Success──→ Supprimer source
                                         └─Error 404 = skip                  └─Error = skip
```

### Node 1 — Trigger (Schedule / Cron)
- **Schedule recommandé** : cron explicite (`0 15 * * * *`) pour caler une minute précise et désynchroniser des autres workflows.
- « Toutes les X heures » ne garantit pas la minute (compte depuis l'activation).

### Node 2 — Lire le manifeste (texte)
- Récupérer le fichier JSON en **mode texte** (As Binary = OFF).
- ⚠️ GitHub renvoie le contenu **encodé en base64** → décoder à l'étape suivante.

### Node 3 — Décoder + parser (Code)
- Mode `Run Once for All Items`. Décode si base64, éclate en N items.
```javascript
const file = $input.first().json;
const decoded = Buffer.from(file.content, 'base64').toString('utf-8');
return JSON.parse(decoded).map(e => ({ json: e }));
```
- Sortie : 1 item par fichier avec `path`, `category`, `processed_at`.

### Node 4 — Récupérer le fichier source (idempotence) ⭐
- Récupérer chaque fichier en **mode binaire** (As Binary = ON, champ `data`).
- **On Error = Continue (using error output)**.
  - **Success** → fichier existe → contenu prêt pour l'upload.
  - **Error (404)** → déjà archivé/supprimé → skip (sortie error non branchée).
- C'est le **mécanisme d'idempotence** : récupérer sert à la fois de preuve d'existence et de source de contenu. Re-lancer 10× ne crée jamais de doublon.

### Node 5 — Uploader vers l'archive
- Envoyer le binaire (`data`) vers le dossier d'archive (par ID, hors de tout dossier surveillé).
- Nom : `$binary.data.fileName` (évite un champ recréé).
- **On Error = Continue** → upload raté = skip, ne déclenche pas la suppression.

### Node 6 — Supprimer de la source (le garde-fou) 🔒
- Branché **uniquement sur la sortie Success de l'upload**.
- `path` via **pairing** vers le node Parse : `{{ $('Node Parse').item.json.path }}` (car le binaire/upload a remplacé `$json`).
- **On Error = Continue** → suppression ratée = réessai au prochain run.

---

## Les 4 garde-fous (obligatoires)

1. **Supprimer seulement après confirmation d'upload** → Delete branché sur Success de l'upload.
2. **N'archiver que ce qui est dans le manifeste** → fichiers non listés restent intacts.
3. **Archive hors du dossier surveillé par l'ingestion** → pas de boucle infinie.
4. **La suppression source ne re-déclenche pas les workflows aval** → vérifier leurs filtres.

---

## Pièges classiques

| Piège | Parade |
|---|---|
| Contenu manifeste illisible | GitHub renvoie du **base64** → décoder avant `JSON.parse`. |
| « no binary file 'data' found » à l'upload | Node intercalé (Set/Edit Fields) **perd le binaire** → garder l'upload directement après le fetch binaire. |
| Champs d'origine absents après upload | `$json` a changé → **pairing** `{{ $('NodeParse').item.json.x }}`. |
| Minute du cron non garantie | Utiliser une expression cron explicite, pas « toutes les X heures ». |
| Doublons à l'upload sur re-run | L'idempotence est côté **source** (fichier disparu après archivage). Le stockage cible peut accepter des doublons de nom : pas de déduplification là-bas. |
| Rangement par sous-dossier | Un « search dossier » au milieu du flux **perd le binaire** → commencer à plat, catégories en v2. |

---

## Ordre de validation recommandé

1. Tester chaque node isolément.
2. Tester l'**upload** seul (delete non branché) → vérifier dans le stockage.
3. Brancher la **suppression** → vérifier la récupérabilité (archive + historique Git).
4. Tester l'**idempotence** : re-run complet → tout en 404 → 0 archivé, 0 supprimé.
5. **Activer** le workflow.

---

## Vérification d'un run

- **Nominal** : fetch Success = N, Error = 0 ; upload = N ; suppression = N. Fichiers dans l'archive, source vidée.
- **À vide** (déjà archivés) : fetch Success = 0, Error = N ; upload = 0 ; suppression = 0. **Comportement attendu.**
