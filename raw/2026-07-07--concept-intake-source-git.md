---
type: raw
title: "concept-intake-source-git"
source_url: "drive:1mNs2gFitpcgqMX_Z4wfoZYiS2csyMdbG"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
type: wiki
title: "Concept — Intake source → Git (pattern idempotent & blindé)"
status: active
publish: none
vault: ai-automation
brand: null
sources:
  - "raw/2026-06-29--howto-generique-intake-source-vers-git-idempotent-blinde.md"
related:
  - "sop/sop-lumina-intake-et-publish"
  - "concept-archivage-n8n-idempotent"
  - "concept-pipeline-memoire-wiki-git"
  - "synthese-lumina-systeme-reference"
updated: 2026-06-29
---

# Concept — Intake source → Git (pattern idempotent & blindé)

> **Objectif :** drainer une boîte d'entrée (Drive, inbox…), classer chaque fichier via un LLM, l'écrire dans Git (la source de vérité), puis nettoyer la source — sans bloquer et **sans jamais perdre un fichier**.

Voir [[sop/sop-lumina-intake-et-publish]] pour l'implémentation Lumina (Drive → GitHub + Notion).

---

## Pourquoi ce pattern est critique

C'est la **porte d'entrée** de toute la connaissance : sans lui, le wiki ne se remplit pas et la mémoire aval (agents + humains) reste vide. La partie délicate : ne jamais perdre un fichier. On ne le supprime de la source qu'**après avoir confirmé** qu'il est bien dans Git.

Deux catastrophes à éviter :
- **Deadlock** : un fichier déjà présent dans Git fige tout le lot si `On Error = Stop`.
- **Perte de données** : supprimer la source alors que l'écriture Git a échoué.

---

## Le pattern — fin du workflow

```
… (classement LLM) → Create (Git) → Get (vérif, Binary OFF) → IF (sha présent ?)
                                                                 ├─ true  → Delete (source)
                                                                 └─ false → ✗ rien (reste pour retry)
```

**Principe :** on tente l'écriture, **puis on vérifie** la présence dans Git, on **ne supprime de la source que si c'est confirmé**. Qu'il vienne d'être créé OU qu'il existait déjà → confirmé → on nettoie. Sinon (vraie panne) → on garde pour retry.

---

## Workflow complet — 11 nodes

1. **Schedule Trigger** — draine la source à intervalle régulier → backlog + nouveaux.
2. **List source** — liste **tous** les fichiers (pas juste un).
3. **Download** — télécharge chaque fichier.
4. **Extract text** — extrait le texte (pour classer + écrire).
5. **LLM Classify** — classifieur LLM (ex. `gpt-4o-mini`, sortie JSON) : entité / repo / chemin, ou rejeter. *(Éviter les fournisseurs instables en 5xx — ex. Gemini free tier ; préférer OpenAI.)*
6. **Parse** (Code, *Run Once for Each Item*) — parse le JSON : `repo`, `path`, `fileContent`, `fileId`, `reject`.
7. **If Rejected** — accepté → écrire ; rejeté → ignorer.
8. **Create (Git)** — écrit le fichier. **On Error = Continue** (un fichier déjà présent renvoie une erreur « sha » qui ne doit pas bloquer le lot).
9. **Get (Git)** (**As Binary OFF**, On Error = Continue) — **vérifie** la présence. Lit via **pairing** (`{{ $('Parse').item.json.repo }}` / `.path`) car après Create, `$json` a changé de forme.
10. **IF** (`{{ $json.sha }}` *is not empty*) — confirmé dans Git ?
11. **Delete (source)** (sortie **true** uniquement) — supprime la source (`{{ $('Parse').item.json.fileId }}`). Sortie false → reste pour le prochain run.

---

## Règles critiques

| Règle | Explication |
|---|---|
| `On Error = Continue` sur **Create** ET **Get** | Sinon un conflit (fichier existant) fige tout le lot. |
| **As Binary = OFF** sur le Get | Sinon pas de champ `sha` → tout part sur la branche false. |
| **Order = Create → Get → IF → Delete** | Create **juste après** If Rejected (sinon perd `fileContent`). |
| **Pairing** `$('Parse').item.json.*` | Après un node qui change la forme de l'item, récupérer les champs d'origine par référence. |
| **Delete uniquement après confirmation Get** | Jamais avant. |

---

## Erreurs fréquentes

| Symptôme | Cause | Solution |
|---|---|---|
| « sha wasn't supplied » | Créer un fichier qui existe déjà → Stop → deadlock | `On Error = Continue` sur Create |
| Tout part sur la branche False | Get en `As Binary = ON` → pas de `sha` en JSON | Binary OFF |
| `fileContent` undefined sur Create | Get inséré AVANT Create | Create **d'abord**, Get ensuite |
| « No path back to node » en test isolé | Normal en test de node seul | Exécuter le workflow en entier |
| Classifieur instable (503/5xx) | Free tier LLM surchargé | Basculer sur un provider stable (OpenAI) |

---

## Adaptation

Remplacer la source (Drive → autre), l'owner/repo, le classifieur LLM. Le pattern **Write → Verify → Delete-if-confirmed** est universel pour « drainer une source vers un stockage ».
