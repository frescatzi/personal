---
type: raw
title: "POS-GENERIQUE_Decoupage-Workflow-n8n-Orchestrateur-Sous-Workflows_20260704"
source_url: "drive:1eZXZPJ_M7hqFpcAI9CKw-QEpc1itegsg"
captured: 2026-07-07
vault: ai-automation
brand: null
immutable: true
---

# POS GÉNÉRIQUE — Découper un workflow n8n monolithique en orchestrateur + sous-workflows (via API REST interne)

Procédure vérifiée le 04.07.2026 (découpage d'un monolithe de 27 nodes en 1 orchestrateur + 6 sous-workflows, testé end-to-end et publié le jour même). Générique par nature — aucune paire spécifique nécessaire.

## 1. Principe

Un processus lisible = un orchestrateur qui enchaîne des sous-workflows métier via `Execute Workflow`. Chaque sous-workflow reçoit **uniquement ses paramètres** en entrée (trigger `When Executed by Another Workflow` avec `workflowInputs`) et renvoie **exactement 1 item** en sortie. Plus aucune référence croisée `$('Node')` entre blocs — c'est la classe de bugs « No path back to referenced node » éliminée structurellement.

## 2. Procédure

1. **Lire le monolithe par l'API** : dans la console du navigateur de l'onglet n8n connecté, `await fetch('/rest/workflows/<id>', {credentials:'include', headers:{'browser-id': localStorage.getItem('n8n-browserId')}})` → JSON complet (nodes, connections, versionId, parentFolder, tags).
2. **Cartographier les références croisées** : pour chaque node, extraire les motifs `$('NomNode')` de `JSON.stringify(n.parameters)`. Ce qui reste interne à un futur bloc est conservé ; ce qui traverse les blocs devient un paramètre d'entrée.
3. **Découper par étape métier** (jamais par commodité technique) : chaque bloc = un sous-workflow nommé d'après son rôle métier.
4. **Construire chaque sous-workflow programmatiquement** : trigger avec `workflowInputs.values` (le contrat), nodes clonés avec réécriture des références croisées en `$('When Executed by Another Workflow').item.json.<champ>` (remplacements de chaînes sur le JSON des paramètres), sous-graphe de connexions filtré, node Set « Sortie » final avec **`executeOnce: true`** (garantit 1 item). Vérifier « zéro référence non résolue » avant de poster. `POST /rest/workflows` avec `{name, nodes, connections, settings:{executionOrder:'v1'}, active:false, parentFolderId}`.
5. **Reconstruire l'orchestrateur** : trigger d'origine conservé (les données épinglées suivent le nom du node), un node `Execute Workflow` par sous-workflow (structure de paramètres clonée d'un node Execute Workflow existant : `workflowId` resourceLocator + `workflowInputs` ResourceMapper avec `mappingMode:'defineBelow'`, `value`, `schema`), garde-fou `If` après la première étape critique (échec → Set d'erreur), références inter-étapes en **`.first()`** (immunité au nombre d'items). `PATCH /rest/workflows/<id>` avec `versionId` courant.
6. **Vérifier après rechargement** de l'éditeur (jamais sur la foi du PATCH seul), puis test end-to-end avec payload jetable.
7. **Publier les sous-workflows d'abord, l'orchestrateur en dernier** : `POST /rest/workflows/<id>/activate` body `{versionId, name, description}` — n8n refuse de publier un workflow référençant des workflows non publiés.

## 3. Difficultés rencontrées

1. La mutation directe du store Pinia + cmd+s ne sauvegarde rien (n8n ne détecte pas les mutations externes).
2. Un onglet éditeur ouvert **écrase les PATCH API par autosave** (il resauvegarde son état périmé, notamment à l'exécution).
3. Un sous-workflow terminé par un Set après un Merge multi-branches renvoyait N items → mappings `undefined` en aval (appariement d'items impossible).
4. Publication de l'orchestrateur refusée tant que les sous-workflows n'étaient pas publiés.
5. Expressions héritées malformées (sans `=` initial ou `}}` final) produisant du texte brut dans les sorties.

## 4. Solutions implémentées

1. Toute écriture passe par l'API REST (`browser-id` header) — jamais par le store.
2. Rechargement systématique de l'éditeur immédiatement après chaque PATCH, avant toute action UI.
3. `executeOnce: true` sur le node de sortie de chaque sous-workflow + références `.first()` dans l'orchestrateur.
4. Ordre de publication inversé : sous-workflows puis orchestrateur.
5. Audit des expressions du monolithe (recherche des valeurs ne commençant pas par `=`) et réécriture propre.

## 5. Leçons apprises

1. **Un refactoring structurel se fait par l'API avec vérification automatisée, pas à la souris** — et se vérifie par relecture de l'état serveur après rechargement.
2. **L'éditeur ouvert est un écrivain concurrent** : après toute écriture API, recharger avant de toucher l'UI.
3. **Contrat d'interface : 1 item en sortie de chaque sous-workflow** — le garantir mécaniquement (`executeOnce`) plutôt que par convention.
4. Un test « vert » ne prouve que le chemin emprunté : tester explicitement les chemins alternatifs (résultat vide, branche erreur) avec des données dédiées.
