---
type: wiki
title: "SOP générique — Pipeline d'ingestion & publication idempotente (n8n)"
status: active
publish: notion
vault: ai-automation
brand: null
sources: ["raw/2026-06-24--sop-generique-pipeline-source-vers-vues.md"]
related: ["sop/sop-lumina-intake-et-publish", "synthese-lumina-systeme-reference"]
updated: 2026-06-24
---

# SOP générique — Robot d'ingestion & publication idempotente (réutilisable)

> Procédure **générique et réutilisable** (toute marque / tout projet). Remplace les `<placeholders>`. Version concrète Lumina : [[sop/sop-lumina-intake-et-publish]].
> Deux patterns indépendants : **(A) Ingestion** d'une inbox vers un store de fichiers, et **(B) Publication idempotente** d'une source vers une vue (Notion, etc.).

---

## 0. Principes (à respecter quel que soit le projet)

- **Une source de vérité unique** (ex. un repo Git). Tout le reste = **sortie dérivée, régénérable, jamais éditée à la main**.
- **Sorties en parallèle, jamais en chaîne** (modèle hub) : chaque consommateur lit la source par le chemin le plus court.
- **Idempotence** : ré-exécuter un workflow doit reproduire le même état final (pas de doublons, pas de dérive).
- **Garde-fou destructif** : ne jamais supprimer/écraser tant que l'écriture cible n'est pas confirmée.
- **MVP d'abord**, un node à la fois, tester avant d'automatiser.

---

## A. Robot d'ingestion (inbox → classifieur LLM → store de fichiers)

**Pattern** : `Déclencheur → Lister TOUS les fichiers de l'inbox → (par fichier) Télécharger → Extraire le texte → Classer (LLM) → Parser → Aiguiller → Écrire au bon endroit → Supprimer de l'inbox`

1. **Déclencheur** : `Schedule` (drain périodique) ou `Webhook`. Éviter un trigger « événement » seul si on veut aussi vider un **backlog** existant.
2. **Lister l'inbox** : un node « list/search files » (pas un trigger « 1 événement »). C'est lui qui rend le drain **multi-fichiers** (backlog + nouveaux). `Return All = ON`.
3. **Télécharger** : File ID = `{{ $json.id }}` (mode Expression).
4. **Extraire le texte** : node d'extraction adapté au type (texte/PDF…).
5. **Classer (LLM)** : HTTP vers un LLM **fiable** ; demander une **sortie JSON stricte** (`response_format: json_object` côté OpenAI ; mot « JSON » présent dans le prompt). `temperature: 0`. Activer **Retry On Fail**.
6. **Parser** (node Code, **Run Once for Each Item**) : lire la réponse LLM de l'item courant, récupérer les données des nodes amont via `$('NodeName').item.json` (appariement par index), construire chemin + frontmatter, retourner **un seul objet**.
7. **Aiguiller** (IF/Switch) : ex. `reject` → ne pas écrire.
8. **Écrire** (Git/stockage) : **On Error = Stop Workflow**.
9. **Supprimer de l'inbox** : référencer l'ID d'origine via `{{ $('<NodeParser>').item.json.<fileId> }}` (pas `$json.id`, qui contient la réponse du node précédent).
10. **Tester** en manuel, puis **activer**.

## B. Publication idempotente (source → vue, upsert par clé)

**Pattern** : `Déclencheur → Lister les fichiers source → (par fichier) Charger → Parser → construire la charge utile (avec une CLÉ stable, ex. URL source) →` **fork** :
- **créer** la version fraîche dans la vue,
- **chercher par CLÉ** l'ancienne entrée → la **fragmenter** → l'**archiver**.

1. **Filtrer la source** : ne garder que les fichiers à publier ; exclure les artefacts (`README`, `index`, etc.).
2. **Parser** : extraire métadonnées + corps. ⚠️ **un node Code = une seule entrée** (ne jamais y brancher un 2ᵉ flux, sinon items parasites/« fantômes »).
3. **Construire la charge utile** : c'est l'étape qui doit produire la **clé d'identité stable** (ex. l'URL source complète) utilisée pour l'upsert.
4. **Créer** : insérer la nouvelle entrée dans la vue.
5. **Chercher l'existante par CLÉ** : requête filtrée sur la clé. ⚠️ Si le filtre est dans un body JSON, le mettre en **mode Expression + `JSON.stringify`** (sinon `{{ }}` part en texte brut → 0 résultat). Matcher sur la **même valeur** que celle écrite (souvent dispo seulement après l'étape « charge utile »).
6. **Fragmenter** (Split Out) le tableau de résultats, puis **archiver/supprimer** chaque ancienne entrée (opération réversible de préférence : archive > suppression dure).
7. **Tester l'idempotence** : 2 runs consécutifs = même nombre d'entrées.

> Pourquoi l'**upsert par clé** plutôt que « tout vider puis tout recréer » : on ne touche jamais à « toute la vue » → impossible d'effacer l'ensemble par accident, et c'est **compatible déclencheur par fichier** (webhook).

---

## C. Challenges génériques & solutions (leçons transférables)

| Thème | Symptôme | Cause racine | Solution générique |
|---|---|---|---|
| **Expressions** | Filtre/appel renvoie 0 / valeur littérale | `{{ }}` dans un body JSON non évalué | Body en **mode Expression** + `JSON.stringify(objet)` |
| **Identité/clé** | L'upsert ne matche jamais | Clé tronquée/différente de la valeur écrite | Filtrer sur **la même donnée** que celle stockée |
| **Items parasites** | Entrée « fantôme » (titre/clé vide) | Un node Code reçoit **2 flux** en entrée | Garder **une seule entrée** par node Code |
| **Itération** | 1 seul item traité sur N | Code en `Run Once for All Items` avec `.first()` | **Run Once for Each Item** + `$json` + `$('…').item` |
| **Références** | « Referenced node doesn't exist » | Mauvais nom de node | Référencer le **nom exact** |
| **Backlog** | Seul 1 fichier traité, le reste reste | Trigger « 1 événement » | Node **list/search** qui énumère tout l'inbox |
| **Conflit d'écriture** | 4xx type « already exists » | Cible déjà présente | Upsert (update avec clé) ou tolérer l'erreur le temps d'un drain |
| **Fournisseur LLM** | 5xx « high demand » persistant | Palier gratuit surchargé | **Basculer de fournisseur/modèle** + **Retry On Fail** |
| **Suppression dangereuse** | Données perdues | Étape destructive jouée seule | Supprimer **après** écriture confirmée (`On Error = Stop`) |
| **Données figées** | Tests trompeurs | Sortie de node **épinglée**/cache | Dé-épingler, re-fetch |

## D. Checklist de fiabilité (avant d'activer)

- [ ] Le workflow est **idempotent** (2 runs = même état).
- [ ] Toute étape destructive est **après** une écriture confirmée (`On Error = Stop`).
- [ ] Les expressions dans les bodies JSON sont **évaluées** (mode Expression).
- [ ] Les nodes Code itèrent sur **tous** les items (Each Item ou `$input.all()`).
- [ ] Un node Code n'a **qu'une** entrée.
- [ ] La source de vérité reste **régénérable** (on peut tout reconstruire).
- [ ] Tags posés (statut/déclencheur/domaine) ; secrets jamais en clair.

*SOP générique v1.0 — 2026-06-24.*
