---
type: raw
title: "sop-lumina-archive-raw-vers-drive"
source_url: "drive:1F13F-pw9qCPFIJjX6WJSw-s6ipi9ARQm"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "SOP Lumina — Archive Raw→Drive (n8n)"
status: active
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-29--howto-archive-raw-vers-drive-workflow-n8n-lumina.md"
  - "raw/2026-06-29--sop-lumina-archive-raw-vers-drive-n8n.md"
related:
  - "concept-archivage-n8n-idempotent"
  - "synthese-lumina-systeme-reference"
  - "sop/sop-lumina-intake-et-publish"
  - "sop/sop-creer-memoire-agents-humains"
updated: 2026-06-29
---

# SOP Lumina — Archive Raw→Drive (n8n)

> **Workflow :** `LUMINA — Archive — Raw→Drive`
> **Statut v1 :** construite, testée de bout en bout, idempotente (2026-06-29).
> **Environnement :** n8n self-hosté `n8n.aftersunpeople.com` v2.10.4 · Hetzner/Coolify · dossier infra LUMINA · projet Personal.

Pour le pattern générique réutilisable, voir [[concept-archivage-n8n-idempotent]].

---

## 1. Objet

Après que Claude Code (`/ingest`) a compilé `raw/ → wiki/` et mis à jour `_archive_queue.json`, ce workflow :
1. **Lit le manifeste** (`_archive_queue.json`) pour savoir quels fichiers archiver.
2. **Archive chaque fichier brut** dans Google Drive (`Archive-Raw`).
3. **Le supprime de `raw/`** — **seulement après confirmation de l'upload**.

`wiki/` = source de vérité. `raw/` redevient vide. Drive = filet de sécurité (provenance).

---

## 2. Place dans le pipeline Lumina

```
Drive Inbox → (intake n8n) → GitHub raw/ → Claude Code /ingest → wiki/ + _archive_queue.json
   → push (Obsidian Git) → cross-trigger pgvector + Notion
                          → [CE WORKFLOW] vide raw/ vers Drive
```

---

## 3. Décisions de design (actées)

| Question | Décision | Pourquoi |
|---|---|---|
| Déclencheur | **Schedule cron `0 15 * * * *`** | Simple, zéro risque de boucle liée aux pushs. `:15` désynchronise des autres workflows à `:00`. |
| Nettoyage du manifeste | **Non** (idempotence côté source) | Entrées périmées → 404 → skip. Inoffensif et moins de nodes. |
| Arborescence Drive | **v1 : à plat dans `Archive-Raw`** | Un « search dossier » au milieu du flux perd le binaire. Valider d'abord le pipeline + garde-fou. |

---

## 4. Chaîne de nodes (v1 finale)

```
Schedule Trigger
   → Get a file (manifeste, text)
   → Code (décoder + parser)
   → Get raw file (binaire + idempotence)  ──Success──→ Google Drive Upload ──Success──→ GitHub Delete
                                            └─Error 404 = skip               └─Error = skip
```

### Node 1 — Schedule Trigger
- Trigger Interval = **Custom (Cron)**, Expression = `0 15 * * * *`.
- n8n utilise **6 champs** : `[seconde] [minute] [heure] [jour] [mois] [semaine]`. Soit : seconde 0, minute 15, toutes les heures.

### Node 2 — Get a file (lire `_archive_queue.json`)
- GitHub Get ; Resource **File** / Operation **Get** ; Owner `frescatzi` ; Repo `ai-automation` ; File Path `_archive_queue.json` ; **As Binary = OFF** ; Reference `main`.
- Sortie : contenu **encodé en base64** dans le champ `content`.

### Node 3 — Code (décoder + parser le manifeste)
- Mode `Run Once for All Items` :
```javascript
const file = $input.first().json;
const decoded = Buffer.from(file.content, 'base64').toString('utf-8');
const entries = JSON.parse(decoded);
return entries.map(e => ({ json: e }));
```
- Sortie : N items `{ path, category, processed_at }`.

### Node 4 — Get raw file (idempotence + contenu) ⭐
- GitHub Get a file ; `File Path = {{ $json.path }}` ; **As Binary = ON** (champ `data`) ; **On Error = Continue (using error output)**.
- **Success** → fichier existe → contenu binaire prêt.
- **Error (404)** → déjà archivé → skip. Relancer 10× : zéro doublon, zéro plantage.

### Node 5 — Google Drive : Upload
- Resource **File** / Operation **Upload** ; **Input Data Field Name = `data`** ; **File Name = `{{ $binary.data.fileName }}`** ; **Parent Folder by ID = `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`** (= `Archive-Raw`) ; On Error = Continue.

### Node 6 — GitHub : Delete a file (le garde-fou) 🔒
- Connecté **uniquement à la sortie Success de l'Upload** → un fichier n'est supprimé que si son upload a réussi.
- `File Path = {{ $('Code (décoder + parser le manifeste)').item.json.path }}` (pairing — `$json` contient les infos Drive après l'upload).
- `Commit Message = archive: suppression de {{ $('Code (décoder + parser le manifeste)').item.json.path }} (copié dans Drive)`.
- Branch `main` ; On Error = Continue.

---

## 5. Garde-fous

1. **Suppression seulement après confirmation upload** → Delete branché sur Success de l'Upload.
2. **Seuls les fichiers du manifeste sont traités** → `README` et fichiers non listés (ex. `vault: personal`) laissés intacts.
3. **`Archive-Raw` hors de `Lumina Inbox`** → pas de boucle infinie (frères sous `OBSIDIAN`, pas l'un dans l'autre).
4. **La suppression dans `raw/` ne déclenche pas pgvector/Notion** → ces workflows filtrent sur `wiki/`.

---

## 6. Challenges résolus (leçons)

| # | Challenge | Cause | Solution |
|---|---|---|---|
| 1 | 404 au Node 2 | `_archive_queue.json` pas encore poussé | Lancer `/ingest` + push Obsidian Git d'abord. |
| 2 | Contenu illisible | GitHub renvoie en **base64** | `Buffer.from(content, 'base64').toString('utf-8')` avant `JSON.parse`. |
| 3 | « no binary file 'data' found » | Node Set intercalé cassait le binaire | Upload **directement** après le fetch binaire ; nom via `$binary.data.fileName`. |
| 4 | `category`/`path` absents après upload | Binaire **remplace** `$json` | **Pairing** `$('Code …').item.json.x`. |
| 5 | Sous-dossiers par catégorie bloquants | « search dossier » perd le binaire | **v1 à plat**, catégories en v2. |
| 6 | Minute du trigger non garantie | `Hours Between Triggers` compte depuis l'activation | Passer en cron **6 champs** explicite. |

---

## 7. Tests réalisés (preuves v1)

- **Upload** : 13/13 sur Success, tous dans `Archive-Raw`.
- **Delete** : 13 commits de suppression sur `main`. `raw/` vidé ; `README` et fichiers non listés intacts.
- **Idempotence** : re-run complet → Node 4 : Success = 0 / Error = 13 → Upload = 0, Delete = 0.

---

## 8. Infos techniques

- **n8n** : v2.10.4 self-hosté. Node GitHub v1.1, Google Drive v3.
- **Repo** : `frescatzi/ai-automation`, branche `main`. Credential `GitHub account`.
- **Drive** : credential `Google Drive account - Live`. `Archive-Raw` = **ID `16FCAzO6mVsQm8ZyC2HS951OsWV4p9Zgk`**. ⚠️ Inbox interdite = `1gjuB438pW7NKo4A-wbjaud_PH7ijnNUW`.
- **Sécurité** : aucun token dans les conversations — les creds vivent dans n8n.

---

## 9. Opérations courantes

**Run nominal** : Node 4 Success = N, Error = 0 ; Upload = N ; Delete = N commits ; fichiers dans `Archive-Raw` ; `raw/` vidé.

**Run à vide** (déjà archivés) : Node 4 Success = 0, Error = N ; Upload = 0 ; Delete = 0. **Comportement attendu.**

**Incidents :**
| Symptôme | Cause | Action |
|---|---|---|
| Node 2 = 404 | `_archive_queue.json` absent | Lancer `/ingest` + push Obsidian Git. |
| Tout en Error Node 4 | Fichiers absents (déjà archivés) | Normal. |
| Upload « no binary 'data' » | Node intercalé cassant le binaire | Upload directement après fetch ; `$binary.data.fileName`. |
| Suppression sans upload | Garde-fou contourné | Vérifier que Delete part de **Success** de l'Upload. |

---

## 10. Évolutions prévues

- **v2** : rangement par catégorie `Archive-Raw/<category>/`.
- Nettoyage éventuel du manifeste (entrées périmées croissantes à chaque run).
- Activer le workflow en **Active** pour que le cron tourne en production.
