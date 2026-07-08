---
type: raw
title: "POS-GENERIQUE_Ingestion-multi-format-vers-banque-vectorielle"
source_url: "drive:1YEAC2qehm5Jtu3fPxlTXOhLMuluwpph5"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# LUMINA — POS GÉNÉRIQUE — Ingestion multi-format (texte/markdown + PDF/dossier) vers banque vectorielle

Patron réutilisable pour alimenter une banque vectorielle depuis (a) du texte/markdown fourni, ou (b) un dossier de fichiers (récursion), avec chunking + embeddings.

## 1. Deux entrées, un même cœur

Cœur commun : **Chunk → Embed → Build Insert (paramétré) → Postgres Insert**.
- **Chunk** : découpe le texte (`size`/`overlap` fixes, ex. 3000/200), propage les métadonnées (brand, title, source_ref, collection).
- **Embed** : 1 appel modèle d'embedding par chunk (même modèle partout).
- **Build Insert** : aligne les embeddings avec les chunks **par index** (`$input.all()` ↔ `$('Chunk').all()`), résout la table cible via un registre/`<code>_memory`, INSERT **paramétré** (`$1..$n`, hash de contenu, `ON CONFLICT DO NOTHING`).

## 2. Entrée A — texte/markdown fourni

`Trigger {brand,title,content,collection,source_ref}` → cœur. Appelable comme outil/sous-workflow (ingérer un doc converti, une note, etc.).

## 3. Entrée B — dossier de fichiers (récursion)

`Trigger {folderId}` → **Search** (lister le dossier) → **Filter** (ne garder que les types supportés : PDF, markdown, texte) → **Download** → **Switch par type** → **Extract** (branche PDF / branche texte) → normaliser en `content` → cœur.

## 4. Garde-fous & pièges

1. **Formats non supportés nativement** (ex. .docx) → convertir en amont (md/pdf) ou exclure ; ne pas tenter de parser ce que le moteur ne lit pas.
2. **Perte de métadonnées après un node HTTP/Extract** (il remplace le json) → ré-aligner via référence indexée au node Chunk/Filter.
3. **Idempotence** via hash de contenu + `ON CONFLICT DO NOTHING` ; éviter `TRUNCATE` si la banque contient d'autres collections (épisodes, skills, insights).
4. **Récursion** : une requête « in parents » ne descend pas dans les sous-dossiers → boucler pour une récursion profonde.
5. **Embedding unique** partout (écriture, recherche) sinon distances incohérentes.

## 5. Tester

Ingérer un fichier connu → vérifier le count par `source_ref` → rechercher un fait du document via le point de récupération (doit remonter avec un bon score).

---
*POS GÉNÉRIQUE — 2026-07-02. Version exacte : POS-AFTRSN ingestion texte + récursion Drive. Brique du LUMINA-PLAYBOOK.*
