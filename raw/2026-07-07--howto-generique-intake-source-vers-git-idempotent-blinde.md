---
type: raw
title: "HOWTO_GENERIQUE_intake-source-vers-git-idempotent-blinde"
source_url: "drive:1GC50chiE-GfU3MV_2FyBY70UPByaxeUv"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

---
title: "How-To (générique) — Intake : Source → Git (idempotent & blindé)"
type: howto
status: active
tags: [n8n, git, intake, idempotence, llm-classifier, generic]
version: 1.0
last_review: 2026-06-25
---

# How-To (générique) — Intake : Source → Git (idempotent & blindé)

> Version **générique et réutilisable** (aucune marque, aucun identifiant réel). Remplace les `<placeholders>`.

## Objectif
Faire entrer automatiquement de la connaissance : prendre les fichiers déposés dans une **source d'entrée** (ex. un dossier cloud surveillé), les **classer**, les écrire dans un **dépôt Git** (la source de vérité), puis **nettoyer la source** — sans bloquer et **sans jamais perdre un fichier**.

## À quoi ça sert / pourquoi c'est important
C'est la **porte d'entrée** de toute la connaissance : sans elle, le wiki ne se remplit pas et la mémoire en aval (agents + humains) reste vide. Le cœur du travail est fait par un **classifieur LLM** qui décide, pour chaque fichier, où il va (entité / dépôt / chemin) ou s'il faut l'ignorer. La partie délicate — et la raison de ce How-To — c'est de **ne jamais perdre un fichier** : on ne le supprime de la source qu'**après avoir confirmé** qu'il est bien dans Git. Ce « blindage » évite deux catastrophes : le **deadlock** (un fichier déjà présent qui fige tout le lot) et la **perte de données** (supprimer la source alors que l'écriture a échoué). Une fois ce workflow solide, la connaissance entre toute seule et la chaîne mémoire prend le relais.

## Quand utiliser
Tout pipeline « drainer une boîte d'entrée → écrire ailleurs → nettoyer la source » où un même fichier peut être re-traité.

## Pré-requis
- n8n avec credentials : **source** (cloud/dossier), **LLM de classification**, **Git**.
- Une boîte d'entrée dédiée.
- Un ou plusieurs **dépôts Git cibles**.

## Le pattern (la fin du workflow)
```
… (classement) → Create (Git) → Get (vérif) → IF (présent ?)
                                                 ├─ true  → Delete (source)
                                                 └─ false → ✗ rien (reste dans la source)
```
**Principe :** on tente l'écriture ; **puis on vérifie** la présence dans Git ; on **ne supprime de la source que si c'est confirmé**. Qu'il vienne d'être créé OU qu'il existait déjà → confirmé → on nettoie. Sinon (vraie panne) → on garde pour retry.

---

## Workflow complet : « Intake — Router (Source→Git) »
Rôle global : draine la boîte d'entrée, classe chaque fichier, l'écrit dans le bon dépôt Git, puis supprime la source en sécurité.

**Chaîne et rôle de chaque node :**
1. **Schedule Trigger** — lance à intervalle régulier → la source se draine seule.
2. **List source** — liste **tous** les fichiers de la boîte (backlog + nouveaux), pas juste un.
3. **Download** — télécharge chaque fichier.
4. **Extract text** — extrait le **texte** (pour classer + écrire).
5. **LLM Classify** — le **classifieur** (ex. un modèle léton type *small/mini*, sortie **JSON**) décide entité / dépôt / chemin, accepter ou rejeter. *(Choisir un fournisseur stable ; éviter ceux qui saturent en 5xx.)*
6. **Parse** (Code, *Run Once for Each Item*) — parse le JSON en champs : `repo`, `path`, `fileContent`, `fileId` (id du fichier source), `reject`…
7. **If Rejected** — aiguille : **accepté** → on écrit ; **rejeté** → on laisse tomber.
8. **Create (Git)** — écrit le fichier dans le dépôt. **Settings → On Error = Continue** (un fichier déjà présent renvoie une erreur qu'il ne faut pas laisser bloquer le lot).
9. **Get (Git)** (**As Binary OFF**, On Error = Continue) — **vérifie** la présence. Lit l'original via pairing (`{{ $('Parse').item.json.repo }}` / `.path`) car après Create `$json` a changé de forme.
10. **IF** (`{{ $json.sha }}` *is not empty*) — **confirmé dans Git ?** true = oui.
11. **Delete (source)** (sur la sortie **true**) — supprime de la boîte **seulement si confirmé** (`{{ $('Parse').item.json.fileId }}`). Sortie **false** non branchée → reste pour un prochain run.

> **Re-publier / réactiver** le workflow après modif.

---

## Bonnes pratiques
- **Ordre = Create → Get → IF → Delete.** Create **juste après** « If Rejected » (sinon il perd `fileContent`).
- **Dépôt cible dynamique** (`{{ $json.repo }}`) si les fichiers vont dans plusieurs dépôts.
- **Pairing `$('Parse').item.json.*`** pour les champs d'origine après un node qui change la forme de l'item.
- **Delete uniquement après confirmation Get** (sortie true).
- Tester les **deux branches** (fichier existant + fichier neuf).

## Erreurs fréquentes
- **Erreur « identifiant manquant » à la création** → on *crée* un fichier qui *existe déjà*. Solution : On Error = Continue + vérif Get.
- **Deadlock en boucle** : Create en `On Error = Stop` → un conflit fige tout le lot. Solution : Continue.
- **« first argument must be a string… undefined »** sur Create → contenu vide, car le Get inséré AVANT Create. Solution : Create d'abord.
- **Tout part sur False** alors que les fichiers existent → Get en **As Binary = ON** (pas d'`sha`/métadonnée en JSON). Solution : Binary OFF.
- **« No path back to node »** en test isolé → normal ; se résout en exécution complète.

## Points critiques
- Ne **jamais** supprimer de la source sans **confirmation** (sinon perte si l'écriture a échoué).
- `On Error = Continue` sur **Create** ET **Get**.
- `As Binary = OFF` sur le Get.

## Adaptation
Remplacer la source, l'owner/dépôt, le classifieur. Le pattern **Write → Verify → Delete-if-confirmed** est universel pour « drainer une source vers un stockage ».

## Version
v1.0 — 2026-06-25 (générique)
