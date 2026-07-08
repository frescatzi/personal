---
type: wiki
title: SOP — Réparer un webhook n8n (last-node) + ingérer un PDF Drive dans une banque vectorielle
status: active
publish: notion
vault: ai-automation
brand:
sources:
  - raw/2026-07-02--lumina-marche-a-suivre-generique-reparer-une-recuperation-webhook-n8n-last-node-ingerer-un-pdf-google-drive-dans-une-banque-vectorielle.md
  - raw/2026-07-02--lumina-pos-generique-runbook-ingestion-pdf-drive-vers-banque-vectorielle-recuperation-webhook-last-node.md
related:
  - wiki/sop/sop-diagnostiquer-pipeline-memoire-vectorielle.md
  - wiki/sop/sop-ingestion-multi-format-banque-vectorielle.md
  - wiki/concept-memoire-vectorielle-multi-marques.md
  - wiki/concept-memoire-vivante-agents.md
updated: 2026-07-06
---

# SOP — Réparer un webhook n8n (last-node) + ingérer un PDF Drive dans une banque vectorielle

## Portée

Patron réutilisable : pipeline RAG n8n (embeddings → pgvector) exposé par webhook/MCP, avec ingestion d'un document Drive (PDF ou texte).

## A. Réparer un webhook qui renvoie n'importe quoi

### 1. Qualifier la panne

| Symptôme | Cause probable |
|----------|---------------|
| `NodeApiError` | Un node interne plante |
| Réponse = sortie d'un node intermédiaire (ex. SQL construit, pas les résultats) | Chaîne coupée en aval de ce node |
| « No Respond to Webhook node found » | Mode `responseNode` actif mais le flux n'atteint jamais de Respond valide |

### 2. Trouver le point de coupure

1. Executions → ouvrir l'exécution rouge → lire la **sortie réelle**.
2. Comparer sortie attendue vs réelle : le **dernier node ayant produit la sortie** = bout vivant ; le lien vers le suivant est rompu.
3. **Ne pas se fier au visuel du canvas** : une connexion peut paraître présente mais être rompue à l'exécution.

### 3. Reconnecter + rendre robuste

- Recréer la connexion rompue (drag sortie → entrée du node suivant).
- Passer en mode **last-node** (sans dépendance à `Respond to Webhook`) :
  - Node `Webhook` : `Respond` = **When Last Node Finishes**, `Response Data` = **All Entries**.
  - **Supprimer** tout node `Respond to Webhook` résiduel (même désactivé, il déclenche « Unused Respond to Webhook node found »).
  - Le **dernier node** de la chaîne (ex. `Postgres Search`) renvoie directement le JSON.
- **Publish** → valider par l'appel réel (MCP/webhook).

## B. Ingérer un PDF Google Drive dans une banque vectorielle

### 4. Inspecter l'arborescence Drive AVANT de coder

- Lister le dossier source. Si la racine ne contient que des **sous-dossiers**, une requête plate `'<folderId>' in parents` ne verra **pas** les fichiers profonds.
- MVP recommandé : **fichier unique par ID** (simple, fiable). Récursion pour les lots.

### 5. Chaîne d'ingestion — fichier unique

```
(TRUNCATE si reload complet)
→ Google Drive Download (By ID, sortie binaire = "data")
→ Extract from File (opération PDF → "Extract From PDF" ; texte → "Extract from text file" ; Input Binary Field = "data")
→ Set Document (Manual Mapping)
→ Chunk → Embed → Build Insert (paramétré) → Postgres Insert → Verify
```

Champs à mapper dans **Set Document** :
- `document` = `{{ $json.text }}`
- `title`, `source` = `Drive`, `source_ref` (traçabilité), `collection`, `knowledge_type`

### 6. Boucle multi-fichiers / types mixtes

- Multi-fichiers : `Google Drive Search` → boucle → `Download (By ID = {{ $json.id }})` → `Extract` → `Set Document`.
- Types mixtes (PDF + .md) : `Switch par mimeType` → branche **Extract From PDF** / branche **Extract from text file** → merger avant `Set Document`. Un node Extract = un seul type.

### 7. Valider end-to-end

- Exécuter → vérifier le `count` par `source_ref`.
- Re-tester par l'**appel MCP réel** (`search_brand_memory(brand=…)`) — pas seulement au niveau du node.

## C. Pièges récurrents

- **Réponse = sortie intermédiaire** → connexion coupée en aval ; recréer le lien (ne pas se fier au canvas).
- **Webhook robuste** → mode « last node » + **aucun** node Respond résiduel.
- **Dossier Drive imbriqué** → requête plate `in parents` ne voit que les sous-dossiers → cibler par ID (MVP) ou boucle récursive.
- **Drive → texte** : `Download (binary 'data')` → `Extract from File` (type adapté) → `$json.text`.
- **Expressions n8n** → auto-close `{{ }}` → nettoyer le `}}` en trop (End + 3 Backspace).
- **PDF design** (peu de texte extractible) → prévoir un `.md`/`.docx` converti si le contenu doit être propre.
- **SQL concaténé** → toujours paramétré (voir [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]]).

## Voir aussi

- [[sop/sop-diagnostiquer-pipeline-memoire-vectorielle]] — diagnostic + SQL paramétré.
- [[sop/sop-ingestion-multi-format-banque-vectorielle]] — ingestion texte + PDF + récursion complète.
- [[concept-memoire-vectorielle-multi-marques]] — schéma banque, registre, multi-marques.
- [[concept-memoire-vivante-agents]] — primitives WRITE/READ/CONSOLIDATE.
